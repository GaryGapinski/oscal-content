SHELL:=/usr/bin/env bash
SRC_DIR:=../src
SRC_FILES:=$(shell find $(SRC_DIR) -name '*.xml' -o -name '*.json')
SRC_XML_PROFILES:=$(shell find $(SRC_DIR) -name '*profile.xml')
SRC_READMES:=$(shell find $(SRC_DIR) -iname 'README.md')
GEN_CONTENT_DIR:=generated
GEN_READMES:=$(patsubst $(SRC_DIR)/%,$(GEN_CONTENT_DIR)/%,$(SRC_READMES))
GEN_XML_FILES:=$(patsubst $(SRC_DIR)/%,$(GEN_CONTENT_DIR)/%,$(SRC_FILES))
GEN_XML_PROFILES:=$(subst _profile.xml,-resolved-profile_catalog.xml,$(patsubst $(SRC_DIR)/%,$(GEN_CONTENT_DIR)/%,$(SRC_XML_PROFILES)))
GEN_JSON_FILES:=$(subst xml,json,$(GEN_XML_FILES))
GEN_MIN_JSON_FILES:=$(subst .json,-min.json,$(subst xml,json,$(GEN_XML_FILES)))
GEN_YAML_FILES:=$(subst xml,yaml,$(GEN_XML_FILES))
CURL_INSTALL_OPTS:=--silent --location
JQ_BIN:=jq-linux-amd64
JQ_PATH:=$(shell pwd)/jq
JQ_INSTALL_VERSION:=jq-1.7
JQ_DOWNLOAD_URL:=https://github.com/jqlang/jq/releases/download/$(JQ_INSTALL_VERSION)/$(JQ_BIN)
JQ_INSTALL_COMMAND=curl $(CURL_INSTALL_OPTS) -o $(JQ_PATH) $(JQ_DOWNLOAD_URL) && chmod +x $(JQ_PATH)
XMLLINT_PATH=/usr/bin/xmllint
XMLLINT_INSTALL_COMMAND=sudo rm -rf /var/lib/apt/lists/* && sudo apt-get update && sudo apt-get install libxml2-utils
YQ_BIN:=yq_linux_amd64
YQ_PATH:=$(shell pwd)/yq
YQ_INSTALL_VERSION:=v4.35.2
YQ_DOWNLOAD_URL:=https://github.com/mikefarah/yq/releases/download/$(YQ_INSTALL_VERSION)/$(YQ_BIN)
YQ_INSTALL_COMMAND:=curl $(CURL_INSTALL_OPTS) -o $(YQ_PATH) $(YQ_DOWNLOAD_URL) && chmod +x $(YQ_PATH)
NPM_PREFIX_DIR:=oscal/build
NPM_PKGS_DIR:=node_modules
XSLT_RUNNER:=oscal/build/xslt-runner.sh
PROFILE_RESOLVER_RUNNER:=oscal/src/utils/resolver-pipeline/oscal-profile-resolve.sh
PROFILE_RESOLVER_ARGS:="hide-source-profile-uri=true" "uuid-method='random-xslt'"
XML_JSON_CONVERTER_XSLT:=oscal/build/generated/oscal_complete_xml-to-json-converter.xsl
OSCAL_COMPLETE_XML_SCHEMA:=oscal/build/generated/oscal_complete_schema.xsd
OSCAL_COMPLETE_JSON_SCHEMA:=oscal/build/generated/oscal_complete_schema.json

.PHONY: help
# Run "make" or "make help" to get a list of user targets
# Adapted from https://marmelab.com/blog/2016/02/29/auto-documented-makefile.html
help: ## Show this help message
	@grep -E '^[a-zA-Z_-]+:.*?##.*$$' $(MAKEFILE_LIST) | awk 'BEGIN { \
	 FS = ":.*?## "; \
	 printf "\033[1m%-30s\033[0m %s\n", "TARGET", "DESCRIPTION" \
	} \
	{ printf "\033[32m%-30s\033[0m %s\n", $$1, $$2 }'

.PHONY: all
all: \
	dependencies             \
	build                    \
	copy-readmes             \
	copy-xml-content         \
	resolve-xml-profiles     \
	convert-min-json-content \
	reformat-json-content    \
	convert-yaml-content     \
	validate-xml-content     \
	validate-json-content    \
	validate-yaml-content ## Run all steps for content preparation

.PHONY: build
build: ## Build core OSCAL artifacts to convert content examples
	$(MAKE) -C oscal/build dependencies
	$(MAKE) -C oscal/build artifacts

.PHONY: dependencies
dependencies: $(JQ_PATH) $(XMLLINT_PATH) $(YQ_PATH) ## Install binary build $(JQ_PATH) and yq dependencies for repo

$(JQ_PATH):
	@$(JQ_INSTALL_COMMAND)

$(XMLLINT_PATH):
	@$(XMLLINT_INSTALL_COMMAND)

$(YQ_PATH):
	@$(YQ_INSTALL_COMMAND)

.PHONY: copy-readmes
copy-readmes: $(GEN_READMES) ## Copy README files to release location

$(GEN_CONTENT_DIR)/%.md: $(SRC_DIR)/%.md
	@mkdir -p $(@D)
	@cp $(SRC_DIR)/$*.md $(GEN_CONTENT_DIR)/$*.md

.PHONY: copy-xml-content
copy-xml-content: $(GEN_XML_FILES) ## Copy OSCAL XML files to release location

$(GEN_CONTENT_DIR)/%.xml: $(SRC_DIR)/%.xml
	@mkdir -p $(@D)
	@cp $(SRC_DIR)/$*.xml $(GEN_CONTENT_DIR)/$*.xml

.PHONY: resolve-xml-profiles
resolve-xml-profiles: $(GEN_XML_PROFILES) ## Resolve OSCAL XML profiles for custom catalogs

$(GEN_CONTENT_DIR)/%-resolved-profile_catalog.xml: $(SRC_DIR)/%_profile.xml
	mkdir -p $(@D)
	$(PROFILE_RESOLVER_RUNNER) $(SRC_DIR)/$*_profile.xml $(GEN_CONTENT_DIR)/$*-resolved-profile_catalog.xml $(PROFILE_RESOLVER_ARGS)

.PHONY: validate-xml-content
validate-xml-content: $(GEN_XML_FILES) ## Validate XML files
	@xmllint --schema $(OSCAL_COMPLETE_XML_SCHEMA) --noout $(GEN_XML_FILES)

.PHONY: convert-min-json-content
convert-min-json-content: $(GEN_MIN_JSON_FILES) ## Convert examples from OSCAL XML to JSON

.SECONDEXPANSION:
$(GEN_CONTENT_DIR)/%-min.json: $(GEN_CONTENT_DIR)/$$(subst json,xml,%).xml
	@mkdir -p $(@D)
	$(XSLT_RUNNER) $(XML_JSON_CONVERTER_XSLT) $(GEN_CONTENT_DIR)/$(subst json,xml,$*).xml $(GEN_CONTENT_DIR)/$*-min.json

.PHONY: reformat-json-content
reformat-json-content: $(GEN_JSON_FILES) ## Format minified JSON to pretty-printed JSON

$(NPM_PREFIX_DIR)/$(NPM_PKGS_DIR):
	$(MAKE) -C oscal/build dependencies

$(GEN_CONTENT_DIR)/%.json: $(GEN_CONTENT_DIR)/%-min.json
	$(JQ_PATH) . $(GEN_CONTENT_DIR)/$*-min.json > $(GEN_CONTENT_DIR)/$*.json

.PHONY: validate-json-content
validate-json-content: $(GEN_JSON_FILES) ## Validate JSON files
	npx --prefix $(NPM_PREFIX_DIR) ajv validate -s $(OSCAL_COMPLETE_JSON_SCHEMA) -c ajv-formats $(foreach file,$(GEN_JSON_FILES),-d $(file))

.PHONY: convert-yaml-content
convert-yaml-content: $(GEN_YAML_FILES) ## Convert examples from OSCAL JSON to YAML

.SECONDEXPANSION:
$(GEN_CONTENT_DIR)/%.yaml: $(GEN_CONTENT_DIR)/$$(subst yaml,json,%).json
	@mkdir -p $(@D)
	@cat $(GEN_CONTENT_DIR)/$(subst yaml,json,$*).json | $(YQ_PATH) e -P - > $(GEN_CONTENT_DIR)/$(subst json,yaml,$*).yaml

.PHONY: validate-yaml-content
validate-yaml-content: $(GEN_YAML_FILES) ## Validate YAML files
	npx --prefix $(NPM_PREFIX_DIR) ajv validate -s $(OSCAL_COMPLETE_JSON_SCHEMA) -c ajv-formats $(foreach file,$(GEN_YAML_FILES),-d $(file))

.PHONY: clean
clean: clean-readmes clean-json-content clean-xml-content clean-yaml-content ## Clean all generated content

.PHONY: clean-readmes
clean-readmes: ## Clean generated README files
	@find $(GEN_READMES)

.PHONY: clean-json-content
clean-json-content: ## Clean generated JSON content
	@rm -f $(GEN_JSON_FILES)
	@rm -f $(GEN_MIN_JSON_FILES)

.PHONY: clean-xml-content
clean-xml-content: ## Clean generated XML content
	@rm -f $(GEN_XML_FILES)

.PHONY: clean-yaml-content
clean-yaml-content: ## Clean generated YAML content
	@rm -f $(GEN_YAML_FILES)


# By default we install xmllint with operating system package manager, so
# to be sensible, we will not uninstall or delete it even with the package
# manager and reduce the amount of friction.
.PHONY: clean-dependencies
clean-dependencies: ## Clean binary dependencies for repo
	@rm -f $(JQ_PATH) $(YQ_PATH)