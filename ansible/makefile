# Makefile to run Ansible with vaulted SSH key and venv

VAULT_KEY=secrets/id_rsa
INVENTORY=inventories/hosts
PLAYBOOK=playbook.yml
VENV=.venv

.PHONY: help install run reset traefik argocd argocd-password kubeconfig tunnel dns

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

tunnel: install ## Ouvrir un tunnel SSH vers l'API server (port 6443) - laisser tourner
	@bash -c '\
		eval "$$(ssh-agent -s)" >/dev/null; \
		. $(VENV)/bin/activate; \
		$(VENV)/bin/ansible-vault view --ask-vault-pass $(VAULT_KEY) | ssh-add - 2>/dev/null; \
		KUBE1=$$(awk "/^\[kube1\]/{getline; print; exit}" $(INVENTORY)); \
		echo "Tunnel SSH actif vers $$KUBE1:6443 -> localhost:6443"; \
		echo "Laisser ce terminal ouvert. Ctrl+C pour fermer."; \
		ssh -N -L 6443:localhost:6443 ec2-user@$$KUBE1; \
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

dns: ## Afficher la commande pour configurer le DNS local (/etc/hosts)
	@INGRESS_HOST=$$(awk '/^\[workers\]/{getline; print; exit}' $(INVENTORY)); \
	INGRESS_IP=$$(dig +short $$INGRESS_HOST 2>/dev/null || nslookup $$INGRESS_HOST 2>/dev/null | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | tail -1); \
	echo ""; \
	echo "IP du node ingress ($$INGRESS_HOST) : $$INGRESS_IP"; \
	echo ""; \
	echo "Executez la commande correspondant a votre OS :"; \
	echo ""; \
	echo "  Mac     : sudo sed -i '' '/kubequest.local/d' /etc/hosts && echo \"$$INGRESS_IP argocd.kubequest.local\" | sudo tee -a /etc/hosts"; \
	echo "  Linux   : sudo sed -i '/kubequest.local/d' /etc/hosts && echo \"$$INGRESS_IP argocd.kubequest.local\" | sudo tee -a /etc/hosts"; \
	echo "  Windows : powershell -Command \"(Get-Content C:\Windows\System32\drivers\etc\hosts | Where-Object { \$$_ -notmatch 'kubequest.local' }) + '$$INGRESS_IP argocd.kubequest.local' | Set-Content C:\Windows\System32\drivers\etc\hosts\" (run as Admin)"; \
	echo ""

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
