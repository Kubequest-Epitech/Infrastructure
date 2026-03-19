# Makefile to run Ansible with vaulted SSH key and venv

VAULT_KEY=secrets/id_rsa
INVENTORY=inventories/hosts
TARGET=servers
PLAYBOOK=playbook.yml
VENV=.venv

.PHONY: help run

help: ## Afficher cette aide
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-22s\033[0m %s\n", $$1, $$2}'

run: ## Launch the playbook
	@echo "Starting ssh-agent with venv..."
	@bash -c '\
		eval "$$(ssh-agent -s)"; \
		. $(VENV)/bin/activate; \
		echo "Adding vaulted SSH key..."; \
		ansible-vault view $(VAULT_KEY) | ssh-add -; \
		echo "Running Ansible playbook..."; \
		ansible-playbook -i $(INVENTORY) $(PLAYBOOK); \
	'