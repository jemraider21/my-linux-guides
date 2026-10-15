# Fish Shell Workflow: `git-just-install`

A helper function for the Fish shell that clones a Git repository, automatically navigates into the cloned directory, and executes build and installation steps via [`just`](https://github.com/casey/just).

---

## Overview

In Fish shell, commands that require argument interpolation and dynamic string manipulation (such as extracting directory names from arbitrary URLs) are best implemented as **functions** rather than standard aliases.

This function performs the following steps:
1. Validates that a repository URL was supplied.
2. Extracts the project folder name (stripping trailing slashes and any `.git` suffix).
3. Clones the repository using `git clone`.
4. Changes directory (`cd`) into the target repository.
5. Executes `just build-release` followed by `sudo just install` sequentially (halting on errors).

---

## Function Implementation

```fish
function git-just-install -d "Clone repo, cd into directory, build and install with just"
    if test (count $argv) -eq 0
        echo "Usage: git-just-install <repo-url>"
        return 1
    end

    set -l repo_url $argv[1]
    
    # Extract directory name by stripping trailing slashes and the .git suffix
    set -l dir_name (string replace -r '/+$' '' $repo_url | path basename | string replace -r '\.git$' '')

    # Step 1: Clone
    git clone $repo_url; or return 1

    # Step 2: Navigate
    cd $dir_name; or return 1

    # Step 3: Build & Install
    just build-release; and sudo just install
    cd ..
end
```

---

## Installation Options

### Option 1: Fish Autoload Directory (Recommended)

Fish natively autoloads functions placed within its functions directory, keeping your `config.fish` clean and improving shell startup time.

1. Create the function file:
   ```fish
   nano ~/.config/fish/functions/git-just-install.fish
   ```
2. Paste the function code into this file and save it.
3. The function will be immediately available in new terminal tabs or windows. To make it available immediately in the current session:
   ```fish
   source ~/.config/fish/functions/git-just-install.fish
   ```

### Option 2: Append to `config.fish`

1. Open your main configuration file:
   ```fish
   nano ~/.config/fish/config.fish
   ```
2. Paste the function block at the end of the file.
3. Reload configuration:
   ```fish
   source ~/.config/fish/config.fish
   ```

---

## Usage

```fish
git-just-install <git-repository-url>
```

### Example

```fish
git-just-install https://github.com/cosmic-utils/clipboard-manager.git
```

---

## Technical Notes

* **Robust Directory Parsing:** Uses Fish's built-in `path basename` and regex substitution (`string replace -r`) to ensure URLs ending with or without trailing slashes or `.git` resolve to the exact directory name created by Git.
* **Error Handling:** Employs `; or return 1` and `; and` guards to halt execution if `git clone` or `just build-release` encounters an error, preventing accidental execution of subsequent commands in the wrong working directory.
