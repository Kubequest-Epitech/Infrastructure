# Makefile to run Ansible with vaulted SSH key and venv

VAULT_KEY=secrets/id_rsa
INVENTORY=inventories/hosts
PLAYBOOK=playbook.yml
VENV=.venv

.PHONY: help install run reset traefik argocd argocd-password kubeconfig

install: ## Installer les dependances Ansible dans le venv
	@if ! $(VENV)/bin/python3 --version >/dev/null 2>&1; then \
		echo "Creating virtual environment..."; \
		rm -rf $(VENV); \
		python3 -m venv $(VENV); \
	fi
	@$(VENV)/bin/pip install -q -r requirements.txt

help: ## Afficher cette aide
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-22s\033[0m %s\n", $$1, $$2}'

run: install ## Launch the playbook
	@echo "Starting ssh-agent with venv..."
	@bash -c '\
		eval "$$(ssh-agent -s)"; \
		. $(VENV)/bin/activate; \
		echo "Adding vaulted SSH key..."; \
		$(VENV)/bin/ansible-vault view --ask-vault-pass $(VAULT_KEY) | ssh-add -; \
		echo "Running Ansible playbook..."; \
		$(VENV)/bin/ansible-playbook -i $(INVENTORY) $(PLAYBOOK); \
	'

traefik: install ## Deployer Traefik sur le cluster
	@echo "Starting ssh-agent with venv..."
	@bash -c '\
		eval "$$(ssh-agent -s)"; \
		. $(VENV)/bin/activate; \
		echo "Adding vaulted SSH key..."; \
		$(VENV)/bin/ansible-vault view --ask-vault-pass $(VAULT_KEY) | ssh-add -; \
		echo "Deploying Traefik..."; \
		$(VENV)/bin/ansible-playbook -i $(INVENTORY) $(PLAYBOOK) --tags traefik; \
	'

argocd: install ## Deployer ArgoCD sur le control-plane
	@echo "Starting ssh-agent with venv..."
	@bash -c '\
		eval "$$(ssh-agent -s)"; \
		. $(VENV)/bin/activate; \
		echo "Adding vaulted SSH key..."; \
		$(VENV)/bin/ansible-vault view --ask-vault-pass $(VAULT_KEY) | ssh-add -; \
		echo "Deploying ArgoCD..."; \
		$(VENV)/bin/ansible-playbook -i $(INVENTORY) $(PLAYBOOK) --tags argocd; \
	'

kubeconfig: install ## Recuperer le kubeconfig du cluster dans ~/.kube/config-kubequest
	@echo "Starting ssh-agent with venv..."
	@bash -c '\
		eval "$$(ssh-agent -s)"; \
		. $(VENV)/bin/activate; \
		echo "Adding vaulted SSH key..."; \
		$(VENV)/bin/ansible-vault view --ask-vault-pass $(VAULT_KEY) | ssh-add -; \
		echo "Fetching kubeconfig..."; \
		$(VENV)/bin/ansible-playbook -i $(INVENTORY) fetch-kubeconfig.yml; \
	'

argocd-password: install ## Recuperer le mot de passe admin ArgoCD
	@bash -c '\
		eval "$$(ssh-agent -s)"; \
		. $(VENV)/bin/activate; \
		$(VENV)/bin/ansible-vault view --ask-vault-pass $(VAULT_KEY) | ssh-add -; \
		$(VENV)/bin/ansible -i $(INVENTORY) kube1 -b -a \
			"kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath={.data.password}" \
			2>/dev/null | tail -1 | base64 -d; \
		echo; \
	'

reset: install ## Supprimer le cluster Kubernetes (kubeadm reset)
	@echo "Starting ssh-agent with venv..."
	@bash -c '\
		eval "$$(ssh-agent -s)"; \
		. $(VENV)/bin/activate; \
		echo "Adding vaulted SSH key..."; \
		$(VENV)/bin/ansible-vault view --ask-vault-pass $(VAULT_KEY) | ssh-add -; \
		echo "Resetting Kubernetes cluster..."; \
		$(VENV)/bin/ansible-playbook -i $(INVENTORY) $(PLAYBOOK) --tags reset; \
	'