# Makefile to run Ansible with vaulted SSH key and venv

VAULT_KEY=secrets/id_rsa
INVENTORY=inventories
HOSTS=inventories/hosts
PLAYBOOK=playbook.yml
VENV=.venv

.PHONY: help install run reset traefik argocd app kyverno namespaces secrets argocd-password kubeconfig tunnel dns

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
		$(VENV)/bin/ansible-playbook -i $(INVENTORY) $(PLAYBOOK) --ask-vault-pass; \
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

app: install ## Redeployer l'app nex (re-apply Application ArgoCD + refresh)
	@echo "Starting ssh-agent with venv..."
	@bash -c '\
		eval "$$(ssh-agent -s)"; \
		. $(VENV)/bin/activate; \
		echo "Adding vaulted SSH key..."; \
		$(VENV)/bin/ansible-vault view --ask-vault-pass $(VAULT_KEY) | ssh-add -; \
		echo "Redeploying nex-application..."; \
		$(VENV)/bin/ansible-playbook -i $(INVENTORY) $(PLAYBOOK) --tags app; \
	'

namespaces: install ## Creer les namespaces depuis inventories/all.yml
	@echo "Starting ssh-agent with venv..."
	@bash -c '\
		eval "$$(ssh-agent -s)"; \
		. $(VENV)/bin/activate; \
		echo "Adding vaulted SSH key..."; \
		$(VENV)/bin/ansible-vault view --ask-vault-pass $(VAULT_KEY) | ssh-add -; \
		echo "Creating namespaces..."; \
		$(VENV)/bin/ansible-playbook -i $(INVENTORY) $(PLAYBOOK) --tags namespaces; \
	'

kyverno: install ## Deployer Kyverno et (re)appliquer les policies
	@echo "Starting ssh-agent with venv..."
	@bash -c '\
		eval "$$(ssh-agent -s)"; \
		. $(VENV)/bin/activate; \
		echo "Adding vaulted SSH key..."; \
		$(VENV)/bin/ansible-vault view --ask-vault-pass $(VAULT_KEY) | ssh-add -; \
		echo "Deploying Kyverno..."; \
		$(VENV)/bin/ansible-playbook -i $(INVENTORY) $(PLAYBOOK) --tags kyverno; \
	'

secrets: install ## Creer les Secrets Kubernetes (donnees sensibles) sur le control-plane
	@echo "Starting ssh-agent with venv..."
	@bash -c '\
		eval "$$(ssh-agent -s)"; \
		. $(VENV)/bin/activate; \
		echo "Adding vaulted SSH key..."; \
		$(VENV)/bin/ansible-vault view --ask-vault-pass $(VAULT_KEY) | ssh-add -; \
		echo "Creating Kubernetes Secrets..."; \
		$(VENV)/bin/ansible-playbook -i $(INVENTORY) $(PLAYBOOK) --tags secrets --ask-vault-pass; \
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
		KUBE1=$$(awk "/^\[kube1\]/{getline; print; exit}" $(HOSTS)); \
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
	@WORKER_IP=$$(awk '/^\[workers\]/{getline; print; exit}' $(HOSTS)); \
	IP=$$(dig +short $$WORKER_IP 2>/dev/null || nslookup $$WORKER_IP 2>/dev/null | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | tail -1); \
	echo ""; \
	echo "Traefik tourne sur tous les workers (DaemonSet)."; \
	echo "N'importe quelle IP worker fonctionne. IP utilisee : $$IP ($$WORKER_IP)"; \
	echo ""; \
	echo "Executez la commande correspondant a votre OS :"; \
	echo ""; \
	echo "  Mac     : sudo sed -i '' '/kubequest.local/d' /etc/hosts && echo \"$$IP argocd.kubequest.local\" | sudo tee -a /etc/hosts"; \
	echo "  Linux   : sudo sed -i '/kubequest.local/d' /etc/hosts && echo \"$$IP argocd.kubequest.local\" | sudo tee -a /etc/hosts"; \
	echo "  Windows : powershell -Command \"(Get-Content C:\Windows\System32\drivers\etc\hosts | Where-Object { \$$_ -notmatch 'kubequest.local' }) + '$$IP argocd.kubequest.local' | Set-Content C:\Windows\System32\drivers\etc\hosts\" (run as Admin)"; \
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
