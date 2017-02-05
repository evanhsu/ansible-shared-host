# Ansible franklinkim.vim role

[![Build Status](https://img.shields.io/travis/weareinteractive/ansible-vim.svg)](https://travis-ci.org/weareinteractive/ansible-vim)
[![Galaxy](http://img.shields.io/badge/galaxy-franklinkim.vim-blue.svg)](https://galaxy.ansible.com/list#/roles/1386)
[![GitHub Tags](https://img.shields.io/github/tag/weareinteractive/ansible-vim.svg)](https://github.com/weareinteractive/ansible-vim)
[![GitHub Stars](https://img.shields.io/github/stars/weareinteractive/ansible-vim.svg)](https://github.com/weareinteractive/ansible-vim)

> `franklinkim.vim` is an [Ansible](http://www.ansible.com) role which:
>
> * installs vim
> * configures vim

## Installation

Using `ansible-galaxy`:

```shell
$ ansible-galaxy install franklinkim.vim
```

Using `requirements.yml`:

```yaml
- src: franklinkim.vim
```

Using `git`:

```shell
$ git clone https://github.com/weareinteractive/ansible-vim.git franklinkim.vim
```

## Dependencies

* Ansible >= 1.6

## Variables

Here is a list of all the default variables for this role, which are also available in `defaults/main.yml`.

```yaml
---
# For more information about default variables see:
# http://www.ansibleworks.com/docs/playbooks_variables.html#id26
#
# vim_config:
#   - "set tabstop=2"
#   - "set shiftwidth=2"
#   - "syntax enable"

# package name (version)
vim_package: vim
# global vim configuration
vim_config: []

```


## Usage

This is an example playbook:

```yaml
---

- hosts: all
  sudo: yes
  roles:
    - franklinkim.vim
  vars:
    vim_config:
      - 'set tabstop=2'
      - 'set shiftwidth=2'
      - 'syntax enable'

```

## Testing

```shell
$ git clone https://github.com/weareinteractive/ansible-vim.git
$ cd ansible-vim
$ vagrant up
```

## Contributing
In lieu of a formal styleguide, take care to maintain the existing coding style. Add unit tests and examples for any new or changed functionality.

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

*Note: To update the `README.md` file please install and run `ansible-role`:*

```shell
$ gem install ansible-role
$ ansible-role docgen
```

## License
Copyright (c) We Are Interactive under the MIT license.
