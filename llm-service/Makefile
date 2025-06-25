# Makefile for llm-service Deployment
# This is a Makefile to ONLY install the llm-service chart.

ifeq ($(NAMESPACE),)
ifeq (,$(filter help,$(MAKECMDGOALS)))
$(error NAMESPACE is not set. Please provide it, e.g., 'make install NAMESPACE=my-namespace')
endif
endif

MAKEFLAGS += --no-print-directory

# --- CONFIGURATION ---
# Default values
HF_TOKEN ?= $(shell bash -c 'read -r -p "Enter Hugging Face Token: " HF_TOKEN; echo $$HF_TOKEN')
LLM_SERVICE_CHART_NAME := llm-service

LLM_SERVICE_CHART_PATH := ./helm

TOLERATIONS_TEMPLATE=[{"key":"$(1)","effect":"NoSchedule","operator":"Exists"}]
# --- END CONFIGURATION ---


# Helm arguments specifically for the llm-service chart
helm_llm_service_args = \
    --set secret.hf_token=$(HF_TOKEN) \
    $(if $(DEVICE),--set device='$(DEVICE)',) \
    $(if $(LLM),--set global.models.$(LLM).enabled=true,) \
    $(if $(SAFETY),--set global.models.$(SAFETY).enabled=true,) \
    $(if $(LLM_TOLERATION),--set-json global.models.$(LLM).tolerations='$(call TOLERATIONS_TEMPLATE,$(LLM_TOLERATION))',) \
    $(if $(SAFETY_TOLERATION),--set-json global.models.$(SAFETY).tolerations='$(call TOLERATIONS_TEMPLATE,$(SAFETY_TOLERATION))',) \
    $(if $(RAW_DEPLOYMENT),--set rawDeploymentMode=$(RAW_DEPLOYMENT),)

# Default target
.PHONY: help
help:
	@echo "Available targets:"
	@echo "  install       - Install the llm-service deployment (creates namespace and deploys Helm chart)"
	@echo "  uninstall     - Uninstall the llm-service deployment and clean up resources"
	@echo "  status        - Check status of the deployment"
	@echo ""
	@echo "Configuration options (set via environment variables or make arguments):"
	@echo "  NAMESPACE                - Target namespace (REQUIRED)"
	@echo "  HF_TOKEN                 - Hugging Face Token (will prompt if not provided)"
	@echo "  DEVICE                   - Deploy models on cpu or gpu (default: gpu)"
	@echo "  LLM                      - Model id as defined in values (eg. llama-3-2-3b-instruct)"
	@echo "  SAFETY                   - Safety model id as defined in values"
	@echo "  LLM_TOLERATION           - Model pod toleration key (e.g., 'nvidia.com/gpu')"
	@echo "  SAFETY_TOLERATION        - Safety model pod toleration key"
	@echo "  RAW_DEPLOYMENT           - Set rawDeploymentMode (e.g., 'true')"
	@echo ""
	@echo "Example Usage:"
	@echo "  make install NAMESPACE=my-llms LLM=llama-3-2-3b-instruct LLM_TOLERATION=\"nvidia.com/gpu\""


# Create namespace and deploy
.PHONY: namespace
namespace:
	@oc create namespace $(NAMESPACE) --dry-run=client -o yaml | oc apply -f -
	@oc config set-context --current --namespace=$(NAMESPACE)

.PHONY: install
install: namespace
	@$(eval LLM_SERVICE_ARGS := $(call helm_llm_service_args))
	@echo "Installing '$(LLM_SERVICE_CHART_NAME)' from path '$(LLM_SERVICE_CHART_PATH)' into namespace '$(NAMESPACE)'..."
	@helm -n $(NAMESPACE) upgrade --install $(LLM_SERVICE_CHART_NAME) $(LLM_SERVICE_CHART_PATH) \
		$(LLM_SERVICE_ARGS)
	@echo "Waiting for llm-service to deploy. This may take some time depending on the model size..."
	@echo "Run 'make status NAMESPACE=$(NAMESPACE)' to check the status of your pods."

# Uninstall the deployment and clean up
.PHONY: uninstall
uninstall:
	@echo "Uninstalling $(LLM_SERVICE_CHART_NAME) helm chart from namespace $(NAMESPACE)..."
	@helm -n $(NAMESPACE) uninstall $(LLM_SERVICE_CHART_NAME)
	@echo "Done."

# Check deployment status
.PHONY: status
status:
	@echo "--- Pods in namespace $(NAMESPACE) ---"
	@oc get pods -n $(NAMESPACE)
	@echo ""
	@echo "--- Services in namespace $(NAMESPACE) ---"
	@oc get svc -n $(NAMESPACE)
	@echo ""
	@echo "--- Secrets in namespace $(NAMESPACE) ---"
	@oc get secrets -n $(NAMESPACE) | grep huggingface-secret || echo "No huggingface-secret found."