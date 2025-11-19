# COPR Makefile for automated Magic Trackpad Monitor builds from GitHub releases
# Builds RPMs from official releases rather than local files

# COPR provides outdir, set default for local testing
outdir ?= .

# Package name and GitHub repository
PACKAGE_NAME = magic-trackpad-monitor
GITHUB_USER = DigitalCyberSoft
GITHUB_REPO = magic-trackpad-monitor

# Default target for COPR
srpm:
	@echo "=== Starting Magic Trackpad Monitor SRPM build from GitHub releases ==="
	@echo "Output directory: $(outdir)"
	@echo "Working directory: $$(pwd)"
	@echo "User: $$(whoami)"

	# Install required build tools if not available
	@if ! command -v git >/dev/null 2>&1; then \
		echo "=== Git not found, installing it ==="; \
		if command -v dnf >/dev/null 2>&1; then \
			echo "Using DNF to install git..."; \
			dnf install -y git || (echo "ERROR: Failed to install git with dnf" && exit 1); \
		elif command -v yum >/dev/null 2>&1; then \
			echo "Using YUM to install git..."; \
			yum install -y git || (echo "ERROR: Failed to install git with yum" && exit 1); \
		else \
			echo "ERROR: No package manager found to install git"; \
			exit 1; \
		fi; \
	fi
	@if ! command -v jq >/dev/null 2>&1; then \
		echo "=== jq not found, installing it ==="; \
		if command -v dnf >/dev/null 2>&1; then \
			echo "Using DNF to install jq..."; \
			dnf install -y jq || (echo "ERROR: Failed to install jq with dnf" && exit 1); \
		elif command -v yum >/dev/null 2>&1; then \
			echo "Using YUM to install jq..."; \
			yum install -y jq || (echo "ERROR: Failed to install jq with yum" && exit 1); \
		else \
			echo "ERROR: No package manager found to install jq"; \
			exit 1; \
		fi; \
	fi

	# Install gcc and X11 development libraries for xidle compilation
	@echo "=== Installing build dependencies ==="; \
	if command -v dnf >/dev/null 2>&1; then \
		dnf install -y gcc || (echo "ERROR: Failed to install gcc" && exit 1); \
		dnf install -y libXScrnSaver-devel || dnf install -y libXss-devel || (echo "ERROR: Failed to install X11 Screensaver libraries" && exit 1); \
	elif command -v yum >/dev/null 2>&1; then \
		yum install -y gcc || (echo "ERROR: Failed to install gcc" && exit 1); \
		yum install -y libXScrnSaver-devel || yum install -y libXss-devel || (echo "ERROR: Failed to install X11 Screensaver libraries" && exit 1); \
	else \
		echo "ERROR: No package manager found to install build dependencies"; \
		exit 1; \
	fi

	# Get latest release version from GitHub API
	@echo "=== Fetching latest Magic Trackpad Monitor release from GitHub ==="; \
	RELEASE_INFO=$$(curl -s https://api.github.com/repos/$(GITHUB_USER)/$(GITHUB_REPO)/releases/latest) || exit 1; \
	VERSION=$$(echo "$$RELEASE_INFO" | jq -r '.tag_name' | sed 's/^v//') || exit 1; \
	if [ -z "$$VERSION" ] || [ "$$VERSION" = "null" ]; then \
		echo "No releases found, checking for tags..."; \
		TAGS_INFO=$$(curl -s https://api.github.com/repos/$(GITHUB_USER)/$(GITHUB_REPO)/tags); \
		VERSION=$$(echo "$$TAGS_INFO" | jq -r '.[0].name' | sed 's/^v//'); \
		if [ -z "$$VERSION" ] || [ "$$VERSION" = "null" ]; then \
			echo "ERROR: Failed to get version from GitHub (no releases or tags found)"; \
			exit 1; \
		fi; \
	fi; \
	echo "Latest Magic Trackpad Monitor version: $$VERSION"; \
	\
	echo "=== Downloading Magic Trackpad Monitor $$VERSION source tarball ==="; \
	curl -L -o $(PACKAGE_NAME)-$$VERSION.tar.gz \
		"https://github.com/$(GITHUB_USER)/$(GITHUB_REPO)/archive/refs/tags/v$$VERSION.tar.gz" || exit 1; \
	\
	echo "=== Extracting tarball to compile xidle ==="; \
	tar xzf $(PACKAGE_NAME)-$$VERSION.tar.gz || exit 1; \
	\
	echo "=== Compiling xidle binary ==="; \
	cd $(PACKAGE_NAME)-$$VERSION && \
	gcc -o xidle xidle.c -lX11 -lXss || (echo "ERROR: Failed to compile xidle" && exit 1); \
	echo "xidle compiled successfully"; \
	cd ..; \
	\
	echo "=== Determining release number ==="; \
	FEDORA_VER=$$(rpm -E %fedora 2>/dev/null || echo "42"); \
	echo "Building for Fedora $$FEDORA_VER"; \
	COPR_USER="$${COPR_USER:-$(GITHUB_USER)}"; \
	COPR_PROJECT="$${COPR_PROJECT:-$(PACKAGE_NAME)}"; \
	COPR_URL="https://download.copr.fedorainfracloud.org/results/$$COPR_USER/$$COPR_PROJECT/fedora-$$FEDORA_VER-x86_64"; \
	BUILD_DIRS=$$(curl -s "$$COPR_URL/" 2>/dev/null | grep -oE '[0-9]+-$(PACKAGE_NAME)' | head -10); \
	EXISTING_RELEASES=""; \
	for dir in $$BUILD_DIRS; do \
		RPMS=$$(curl -s "$$COPR_URL/$$dir/" 2>/dev/null | grep -oE "$(PACKAGE_NAME)-$$VERSION-[0-9]+\.fc[0-9]+\.x86_64\.rpm" | head -1); \
		if [ -n "$$RPMS" ]; then \
			REL=$$(echo "$$RPMS" | sed "s/.*-\([0-9]\+\)\.fc[0-9]\+\.x86_64\.rpm/\1/"); \
			EXISTING_RELEASES="$$EXISTING_RELEASES $$REL"; \
		fi; \
	done; \
	if [ -n "$$EXISTING_RELEASES" ]; then \
		MAX_REL=$$(echo $$EXISTING_RELEASES | tr ' ' '\n' | sort -n | tail -1); \
		RELEASE_NUM=$$(($$MAX_REL + 1)); \
		echo "Found existing release $$MAX_REL, incrementing to $$RELEASE_NUM"; \
	else \
		RELEASE_NUM=1; \
		echo "No existing builds found for $$VERSION, starting with release 1"; \
	fi; \
	\
	echo "=== Recreating source tarball with compiled xidle ==="; \
	tar czf $(PACKAGE_NAME)-$$VERSION.tar.gz $(PACKAGE_NAME)-$$VERSION/ || exit 1; \
	echo "Source tarball created with xidle binary"; \
	\
	echo "=== Generating RPM spec file ==="; \
	sed -e "s/@VERSION@/$$VERSION/g" \
	    -e "s/@PACKAGE_NAME@/$(PACKAGE_NAME)/g" \
	    -e "s/Release:        1%{?dist}/Release:        $$RELEASE_NUM%{?dist}/g" \
	    $(PACKAGE_NAME)-$$VERSION/magic-trackpad-monitor.spec > $(PACKAGE_NAME).spec || exit 1; \
	echo "Spec file generated"; \
	\
	echo "=== Building SRPM ==="; \
	rpmbuild -bs \
		--define "_sourcedir $$(pwd)" \
		--define "_specdir $$(pwd)" \
		--define "_builddir $$(pwd)" \
		--define "_srcrpmdir $(outdir)" \
		--define "_rpmdir $$(pwd)" \
		$(PACKAGE_NAME).spec || exit 1; \
	\
	echo "=== SRPM build complete ==="; \
	ls -la $(outdir)/*.src.rpm

clean:
	@rm -f *.tar.gz *.spec *.src.rpm *.rpm xidle
	@rm -rf $(PACKAGE_NAME)-*/

.PHONY: srpm clean
