Ansible Role Build Images
=========================

This role prepares containers intended for HPC cluster node images in warewulf4. They can then be modified, and finally imported into warewulf and/or exported in to a registry. In the example playbook, the warewulf server is called rhel001 (also set in the variable ww4_host). The playbook installs a few packages and then saves the modified images.


Role Variables
--------------


```
registry_url: "docker.io" # on which server to store the container images


# These variables are overwritten by the example playbook with values from the inventory.
base_container_image:  "docker://ghcr.io/hpcng/warewulf-rockylinux:9"     # start building from this container
container_name:        "myContainer"                                      # container name in podman, used as the "hostname" in ansible
container_image:       "myWw4Container"                                   # save the podman container as this image in podman
container_image_tag:   "1.0"                                              # save the podman container with this tag in podman
ww4_container:         "myImage"                                          # save the podman container as this container in warewulf
ww4_import:            false                                              # import the podman container to warewulf or not?
reg_image:             "myImage"                                          # save the podman container as this image in the container registry or not?
reg_export:            false                                              # export the podman container to the registry


```

Dependencies
------------

 - podman-remote is used to control podman on the ww4 server, so:
   - on the ww4/podman server, one needs to `systemctl start podman.socket`
   - on the ansible server,
     - `dnf install podman-remote`
     - `podman-remote system connection add rhel001 --identity /root/.ssh/id_ed25519 ssh://root@rhel001.triton.aalto.fi/run/podman/podman.sock`
 - podman push assumes a prior `podman login` onto the registry

Example Playbook
----------------

```
---

- name: Prepare a base container
  hosts: rhel001
  remote_user: root


  tasks:


  - name: prepare containers
    ansible.builtin.include_role:
      name: ansible_role_build_images
      tasks_from: prepare_container
    vars:
      container_name: "{{item}}"
      base_container_image: "{{hostvars[item|quote].base_container_image}}"
    loop: "{{groups['ww4_images']}}"


- name: Modify the running container
  hosts: ww4_images

  connection: containers.podman.podman

  vars:
    ansible_podman_executable: "/usr/bin/podman-remote"
    ansible_podman_extra_args: "--connection {{ww4_host}}"

  tasks:

  - name: Install packages to the container
    ansible.builtin.dnf:
      name: "{{packages_to_install}}"
      state: present
      allowerasing: yes


  - name: Install lustre
    ansible.builtin.include_role:
      name: ansible_role_lustre
      tasks_from: install_binary_rpms


- name: Save the modified container
#  hosts: "{{ww4_host}}"
  hosts: rhel001
  remote_user: root

  tasks:

  - name: Save container as ww4 images and/or export to registry
    ansible.builtin.include_role:
      name: ansible_role_build_images
      tasks_from: save_container
    vars:
      container_name:               "{{item}}"
      container_image:     "{{hostvars[item|quote].container_image    }}"
      container_image_tag: "{{hostvars[item|quote].container_image_tag}}"
      ww4_container:       "{{hostvars[item|quote].ww4_container}}"
      ww4_import:          "{{hostvars[item|quote].ww4_import}}"
      reg_export:          "{{hostvars[item|quote].reg_export}}"
      reg_image:           "{{hostvars[item|quote].reg_image}}"
      reg_image_tag:       "{{hostvars[item|quote].reg_image_tag}}"
    loop: "{{groups['ww4_images']}}"

```

Example inventory
-----------------

```
all:
  hosts:
    rhel001
  children:
    ww4_images:
      hosts:

        compute_image_rocky92:
          base_container_image:    "docker://ghcr.io/hpcng/warewulf-rockylinux:9"
          container_image:     "ww4_compute_image_rocky92"
          container_image_tag: "92"
          ww4_container: "compute_image_rocky92"
          ww4_import: true
          reg_image: "{{registry_url}}/compute_image_rocky92"
          reg_image_tag: "1.0"
          reg_export: false
          
        compute_image_rocky92b:
          base_container_image:  "docker://ghcr.io/hpcng/warewulf-rockylinux:9"
          container_image:     "ww4_gpu_image_rocky92"
          container_image_tag: "92"
          ww4_container: "gpu_image_rocky92b"
          ww4_import: false
          reg_image: "{{registry_url}}/gpu_image_rocky92"
          reg_image_tag: "1.0"
          reg_export: true

```


License
-------

BSD

Author Information
------------------

Simppa Äkäslompolo, Aalto Scientific Computing <simppa.akaslompolo@aalto.fi>
