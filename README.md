# pkmaint

An Ansible role that helps package maintainers to build and test their packages
on Fedora based systems.

## Requirements

There are no extra requirements at now.

## Role Variables

```yaml
# Environment variable that contain a path to a file with image specific
# variables:
pkmaint_envvar_imagevarsfile: null

# Variables to be read from file found at location given by enviromnment
# variable named {{ pkmaint_envvar_imagevarsfile }}. These are OS specific,
# so they should be placed to file next to the image.

# OS image distribution (e.g. Fedora):
pkmaint_distro: null

# OS image release version number (e.g. 31):
pkmaint_releasever: null

# OS image architecture (e.g. x86_64):
pkmaint_basearch: null

# mock configuration to be used for building packages (e.g.
# "fedora-{{ pkmaint_releasever }}-{{ pkmaint_basearch }}"):
pkmaint_mock_config: null

# List of dictionaries characterizing OS image related repositories; a keys
# in a dictionary has same meaning as their equivalents from yum_repository
# module; example with all supported keys:
#
#   pkmaint_baserepos:
#     - name: MyRepo
#       description: "My repository"
#       baseurl: "https://repocloud.foo/MyRepo"
#       gpgcheck: no
#       enabled: yes
#
# Repositories can be also installed from packages. To do this, use 'package'
# key:
#
#   pkmaint_baserepos:
#     - package: foo-repos
#
pkmaint_baserepos: []

# List of basic packages (needed for every build), example:
#
#   pkmaint_basepackages:
#     - gcc
#     - gcc-c++
#     - gzip
#     - xz
#
pkmaint_basepackages: []

# Following variables are case specific. They should be placed to playbook.

# List of additional repositories:
pkmaint_customrepos: []

# List of additional packages:
pkmaint_custompackages: []

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

# A name of user that can run mock:
pkmaint_mockuser: null

# A list of subjects for building/testing. One subject describe a package to be
# build, installed, and tested. The structure of subject is
#
#   - name: "package_name"
#     version: "package_version"
#     release: "package_release_number"
#     sources: [] # a list of sources, see below
#     patches: [] # a list of patches
#
# Sources are represented as list of dictionaries, each dictionary provides
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

# A name of local repository:
pkmaint_localrepo: null

# A name of build script that is generated into mock user's home directory:
pkmaint_buildscript: null

# A name of script that updates local repository (created on guest). It is
# installed to /usr/local/bin:
pkmaint_updatelocalreposcript: null
```

## Dependencies

There are no dependencies on other Ansible roles at now.

## Example Playbook

Suppose you have `~/.images/Fedora-31-x86_64.qcow2` image. First, create a file
`~/.images/Fedora-31-x86_64.yml` with image specific variables:
```yaml
pkmaint_distro: Fedora
pkmaint_releasever: 31
pkmaint_basearch: x86_64
pkmaint_mock_config: fedora-{{ pkmaint_releasever }}-{{ pkmaint_basearch }}
pkmaint_basepackages:
  - bash
  - bzip2
  - coreutils
  - cpio
  - diffutils
  - findutils
  - gawk
  - gcc
  - gcc-c++
  - grep
  - gzip
  - info
  - make
  - patch
  - redhat-rpm-config
  - rpm-build
  - sed
  - shadow-utils
  - tar
  - unzip
  - util-linux
  - which
  - xz
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

    pkmaint__envvar_imagevarsfile: IMAGE_VARS_FILE
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

  roles:
    - pkmaint
```

It is supposed that you have also `foo.spec.j2` along with `setup.yml` in your
directory.

Supposing that you have virtual machine with `~/.images/Fedora-31-x86_64.qcow2`
reachable at `127.0.0.3:5555` via SSH with private key `id_rsa`, you can now
run:
```
$ _ssh_opts="-o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"
$ IMAGE_VARS_FILE=~/.images/Fedora-31-x86_64.yml ansible-playbook -vv -u root \
    -i 127.0.0.3, -e "{\"ansible_port\": \"5555\",
    \"ansible_ssh_private_key_file\": \"id_rsa\",
    \"ansible_ssh_common_args\": \"${_ssh_opts}\",
    \"ansible_python_interpreter\": \"/usr/bin/python3\"}" setup.yml
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
# dnf -y install foo
```

Now you can test freshly installed `foo` on your virtual machine. If you change
`foo.spec.j2` or add a next patch, rerun `setup.yml`, `./build`, and
`updatelocalrepo`, update `foo` and test again!

## License

MIT

## Author Information

* Jiří Kučera <sanczes@gmail.com>
