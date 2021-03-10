# pkmaint

An Ansible role that helps package maintainers to build and test their packages
on Fedora based systems.

## Requirements

There are no extra requirements at now.

## Role Variables

```yaml
# General purpose variables
# =========================

# A name of task to be performed. Currently supported tasks are:
#   - set_distinfo
#   - passwordless_sudo
#   - create_users
#   - setup_mock
#   - setup_build
#   - create_local_repo
#   - create_image_setup
pkmaint_task: null

# Task specific variables
# =======================

# set_distinfo task
# -----------------
# Variables:
#   - pkmaint_distinfo

# A dictionary with distribution specific information. It should contains at
# least following keys:
#   - name: A name of the distribution (e.g. Fedora)
#   - release: A release version number of the distribution (e.g. 31)
#   - basearch: An architecture of the distribution (e.g. x86_64)
pkmaint_distinfo: null

# create_users task
# -----------------
# Variables:
#   - pkmaint_users

# List of users to be created. One item can be either string or dictionary.
# Strings are treated as user names, dictionaries can contain following keys:
#
#   name, uid, groups, append
#
# Keys have the same meaning as in user module. Example:
#
#   pkmaint_users:
#     - joe
#     - name: alice
#       groups:
#         - wheel
#       append: yes
#
pkmaint_users: []

# setup_mock task
# ---------------
# Variables:
#   - pkmaint_mockuser
#   - pkmaint_mockprofile_template
#   - pkmaint_mock_profiles

# A name of user that can run mock
pkmaint_mockuser: null

# A name of mock configuration template (without .j2 suffix)
pkmaint_mockprofile_template: null

# List of profiles to be generated to /etc/mock. The structure of one item is
# following:
#
#   name: "a name of profile under /etc/mock, e.g fedora-32-x86_64"
#   template: "a name of mock configuration template (without .j2 suffix)"
#   config_opts:
#     # mock config_opts dict key-value pairs, supported are:
#     chroot_setup_cmd: "..."
#     root: "..."
#     target_arch: "..."
#     legal_host_arches: "( 'arch1', 'arch2', ... )"
#     dist: "..."
#     releasever: "..."
#     bootstrap_image: "..."
#     package_manager: "(default is dnf)"
#     dnf_install_command: "..."
#   package_manager_conf: >
#     a name of package manager config (defaul is dnf.conf)
#   module_platform_id: "module_platform_id value in package_manager_conf"
#   repos:
#     # List of repositories in package_manager_conf. Name of keys coincides
#     # with dnf.conf, see man dnf.conf. An example of one item with all
#     # supported keys is:
#     - name: "..."
#       baseurl: "..."
#       module_hotfixes: true or false
#
pkmaint_mock_profiles: []

# setup_build task
# ----------------
# Prerequisites:
#   - set_distinfo
# Variables:
#   - pkmaint_subjects
#   - pkmaint_mockuser
#   - pkmaint_buildscript
#   - pkmaint_buildscript_template
#   - pkmaint_mock_config

# A list of subjects for building/testing. One subject describe a package to be
# build, installed, and tested. The structure of one subject item is
#
#   - name: "package_name"
#     version: "package_version"
#     release: "package_release_number"
#     sources: [] # a list of sources, see below
#     patches: [] # a list of patches
#
# Sources are represented as a list of dictionaries, each dictionary provides
# information how to get the final source. The structure of one source item is
#
#   - source: "URL, path or other; depends on 'type'"
#     type: 'url' | 'file' | 'command' | 'script'
#     creates: >
#       final result path; if exists, the action of getting source will be
#       skipped
#
# Example:
#
#   pkmaint_subjects:
#     - name: "foo"
#       version: 1.14
#       release: 7
#       sources:
#         - source: "https://filecloud.foo/foo-1.14.tar.gz"
#           type: 'url'
#           creates: "foo-1.14.tar.gz"
#         - source: "mv foo-1.14.tar.gz foo-temp-1.14.tar.gz"
#           type: 'command'
#           creates: "foo-temp-1.14.tar.gz"
#         - source: "add-submodules.sh"
#           type: 'script'
#           creates: "foo-1.14.tar.gz"
#       patches:
#         - foo-1.14-fix-segfault.patch
#
pkmaint_subjects: []

# A name of build script that is generated into mock user's home directory
pkmaint_buildscript: null

# A name of build script template (without .j2 suffix)
pkmaint_buildscript_template: null

# mock configuration to be used for building packages (e.g.
# "fedora-{{ release }}-{{ basearch }}")
pkmaint_mock_config: null

# create_local_repo task
# ----------------------
# Prerequisites:
#   - set_distinfo
# Variables:
#   - pkmaint_localrepo
#   - pkmaint_updatelocalreposcript
#   - pkmaint_updatelocalreposcript_template
#   - pkmaint_mockuser

# A name of local repository
pkmaint_localrepo: null

# A name of script that updates local repository (created on guest). It is
# installed to /usr/local/bin
pkmaint_updatelocalreposcript: null

# A name of template (without .j2 suffix) of script that updates local
# repository
pkmaint_updatelocalreposcript_template: null

# create_image_setup task
# -----------------------
# Variables:
#   - pkmaint_imagebase
#   - pkmaint_image_setup_template

# A name of the qcow2 image without qcow2 extension
pkmaint_imagebase: null

# A name of the template of the image setup playbook. The possible values are:
#   - fedora-setup
#   - rhel-compose-setup
#   - rhel-released-setup
# To tell image setup whether to install mock or not, define with_mock extra
# variable. The default value of with_mock variable differs per image setup,
# more precisely:
#   - fedora-setup: true
#   - rhel-compose-setup: false
#   - rhel-released-setup: true
pkmaint_image_setup_template: null
```

## Dependencies

There are no dependencies on other Ansible roles at now.

## Example Playbook

Suppose you have `~/.images/Fedora-31-x86_64.qcow2` image. First, create an
image setup playbook `~/.images/Fedora-31-x86_64.yml`:
```yaml
- name: Setup image
  hosts: all
  vars:
    distribution: Fedora
    release: 31
    architecture: x86_64

  tasks:
    - name: Setup distribution info
      import_role:
        name: pkmaint
      vars:
        pkmaint_task: set_distinfo
        pkmaint_distinfo:
          name: "{{ distribution }}"
          release: "{{ release }}"
          basearch: "{{ architecture }}"

    - name: Allow passwordless sudo
      import_role:
        name: pkmaint
      vars:
        pkmaint_task: passwordless_sudo

    - name: Setup mock environment
      import_role:
        name: pkmaint
      vars:
        pkmaint_task: setup_mock
```

Next, you can create `setup.yml` playbook:
```yaml
- name: Setup Fedora based system for testing foo
  hosts: all
  vars:
    foo_name: foo
    foo_version: 1.14
    foo_release: 7
    foo_tarball: "{{ foo_name }}-{{ foo_version }}.tar.gz"

  tasks:
    - name: Prepare subjects for building
      import_role:
        name: pkmaint
      vars:
        pkmaint_task: setup_build
        pkmaint_subjects:
          - name: "{{ foo_name }}"
            version: "{{ foo_version }}"
            release: "{{ foo_release }}"
            sources:
              - source: https://filecloud.foo/{{ foo_name }}/{{ foo_tarball }}
                type: url
                creates: "{{ foo_tarball }}"
            patches:
              - foo-1.14-fix_segfault.patch

    - name: Create local repository
      import_role:
        name: pkmaint
      vars:
        pkmaint_task: create_local_repo
```

It is supposed that you have also `foo.spec.j2` along with `setup.yml` in your
directory.

Supposing that you have virtual machine with `~/.images/Fedora-31-x86_64.qcow2`
reachable at `127.0.0.3:5555` via SSH with private key `id_rsa`, you can now
run:
```
$ _ssh_opts="-o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"
$ ansible-playbook -vv -u root \
    -i 127.0.0.3, -e "{\"ansible_port\": \"5555\",
    \"ansible_ssh_private_key_file\": \"id_rsa\",
    \"ansible_ssh_common_args\": \"${_ssh_opts}\",
    \"ansible_python_interpreter\": \"/usr/bin/python3\"}" \
    ~/.images/Fedora-31-x86_64.yml setup.yml
```

After a while, you can SSH to `127.0.0.3:5555` as `mocker` and run `./build` to
build `foo` package:
```
$ ssh -p 5555 ${_ssh_opts} -i id_rsa mocker@127.0.0.3
# ./build
# logout
```

When `foo` builds successfully, you can update local repository with the fresh
build of it and then install it with `dnf`:
```
$ ssh -p 5555 ${_ssh_opts} -i id_rsa root@127.0.0.3
# updatelocalrepo
# dnf --enablerepo local -y install foo
```

Now you can test freshly installed `foo` on your virtual machine. If you change
`foo.spec.j2` or add a next patch, rerun `setup.yml`, `./build`, and
`updatelocalrepo`, update `foo` and test again!

## License

MIT

## Author Information

* Jiří Kučera <sanczes@gmail.com>
