# Ansible plugin for setting environment variables from Gitlab CI

*Note: GitLab is a really great product, and I really recommend companies to go for the EE version. As an individual I can not afford it though, hence this workaround.*


This plugin allows me to define per-environment variables in a Gitlab project CI/CD pipeline.
If for example I have two environments `production` and `staging`, all of which need their own variable `PGDB_PASSWORD`, but a common `PGDB_USERNAME`, I define the following variables in GitLab:

	common_PGDB_USERNAME
	staging_PGDB_PASSWORD
	production_PGDB_PASSWORD

And the ansible deployment role for the production environment will - using this ansible plugin function - create the folling variables:

	PGDB_USERNAME (from common_PGDB_USERNAME)
	PGDB_PASSSWORD (from production_PGDB_PASSWORD)

For the staging environment the following vars will be created:

	PGDB_USERNAME (from common_PGDB_USERNAME)
	PGDB_PASSSWORD (from staging_PGDB_PASSWORD)



## Usage

Drop this file into an ansible [lookup_plugins](http://docs.ansible.com/ansible/devel/plugins/lookup.html) folder. This will result in an ansible function `gitlab_env`. 

How do I use this? I run the following from a .gitlab-ci.yml file deploy section:

    variables:
      COMMON: common
      ENV: ${CI_ENVIRONMENT_SLUG}
    scripts:
    - ansible-playbook product.yml -e env=$ENV -e token=$READ_REGISTRY_TOKEN


This is my product.yml file: 
    
    - name: "Deploy the product"
      hosts: all
      become: true
      roles:
      - { role: stack-deploy }

The stack-deploy role has this plugin in the `lookup_plugins` folder.

The deployment task of the role looks like this:

	- name: Deploy product stack file
	  copy: src=product.yml dest=.
	
	- name: Login to the registry
	  command: docker login -u gitlab -p {{ token }} registry.example.com
	
	- name: Actually deploy the stack
	  command: docker stack deploy -c product.yml product_{{ env }} --with-registry-auth
	  environment: "{{ lookup('gitlab_env', 'ENV', 'COMMON')|combine(item) }}"
	  with_items:
	    - {'SWARM': "{{ lookup('env', 'SWARM') }}" }
	
	- name: remove remote stack file
	  file:
	    path: "product.yml"
	    state: absent
	    
	    
It's the environment line that does the trick. It copies environment variables from the CI environment (including everything that starts with `CI_`, and creates variables that begin with the prefixes defined by 'ENV' and 'COMMON'.