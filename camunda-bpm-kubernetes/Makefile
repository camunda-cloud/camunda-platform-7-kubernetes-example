# @afirth 2019-05

# Make config
.SHELLFLAGS := -eu -o pipefail -c
MAKEFLAGS += --warn-undefined-variables
SHELL = /bin/bash
.SUFFIXES:

.PHONY: all
all: deps
	kustomize build | kubectl apply -f -

.PHONY: dry-run
dry-run: deps
	kustomize build | kubectl apply --dry-run -o yaml -f -

.PHONY: skaffold
skaffold: dev-deps
	skaffold run

.PHONY: deps
deps:
	@which kustomize 1>/dev/null || echo kustomize not found. Try go get sigs.k8s.io/kustomize

.PHONY: dev-deps
dev-deps: deps
	@which skaffold 1>/dev/null || echo skaffold not found. Try https://skaffold.dev/docs/getting-started/#installing-skaffold



