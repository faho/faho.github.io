# fish_vcs_prompt - output version control system information for use in a prompt

## Synopsis

```
function fish_prompt
     printf '%s' $PWD (fish_vcs_prompt) ' $ '
end
```

## Description

The `fish_vcs_prompt` function displays information about the current version control system (VCS) repository, if any.

It calls out to VCS-specific functions. The currently supported systems are:


* fish_git_prompt


* fish_hg_prompt


* fish_svn_prompt

If a VCS isn't installed, the respective function does nothing.

The svn prompt is disabled by default because it's slow on large svn repositories. To enable it, modify fish_vcs_prompt to uncomment it. See funced.

For more information, see the documentation for each of the functions above.

## Example

A simple prompt that displays all known VCS info:

```
function fish_prompt
    ...
    set -g __fish_git_prompt_showupstream auto
    printf '%s %s$' $PWD (fish_vcs_prompt)
end
```
