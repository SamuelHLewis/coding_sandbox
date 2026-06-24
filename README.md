# Coding Sandbox
This is a guide for setting up a sandbox to run a coding assistant inside.

## Do This Once

### Podman Installation
We use Podman because it doesn't require root privileges. To install it on MacOS, type into terminal:
```
brew install podman
```

Then create the lightweight Linux VM that will host your containers:
```
podman machine init
```
You only need to do this once, the machine is ready to start each time to turn on your computer.

### Podman Custom Command
Add one of the functions below to your `.zshrc` or `.bash_profile`. Each follows the same general logic:

1. **Persistent volumes** — a dedicated home directory on your Mac (for credentials/config) and your target workspace are mounted into the container.
2. **Auto-install on first run** — if the CLI binary is missing from the persistent volume, it is downloaded and installed there so subsequent runs skip this step.
3. **Execute** — the binary is added to `PATH` and launched with any arguments you passed.

*Note:* once you have added the function, don't forget to source the config file again so that the command is available to use e.g. for zsh:
```
source ~/.zshrc
```

#### Google Antigravity
Note: this command was valid as of 24/06/2026, but is fragile to changes in the antigravity installation script URL.
```
agy_sandboxed() {
  # this allows the user to specify their working dir when launching the function
  # if not specified, it falls back to the current working dir
  local target_dir="${1:-$(pwd)}"
  shift
  
  # 1. Create persistent config and binary folders on your Mac if they don't exist
  # write permissions need to be explicitly given to the config folder for token refresh
  mkdir -p "$HOME/.ai-sandbox-home/.local/bin"
  mkdir -p "$HOME/.ai-sandbox-home/.gemini"
  chmod 700 "$HOME/.ai-sandbox-home/.gemini"
  
  echo "Starting sandbox for directory: $target_dir"
  
  # 2. Run the container with an auto-installation fallback
  podman run --rm -it \
    -v "$HOME/.ai-sandbox-home:/root:Z" \
    -v "$target_dir:/workspace" \
    -w "/workspace" \
    debian:stable-slim bash -c '
      # always install certificates for authentication signing
      apt-get update && apt-get install -y ca-certificates >/dev/null 2>&1
      # Install agy to the mounted persistent volume if it is missing
      if [ ! -f /root/.local/bin/agy ]; then
        echo "Antigravity CLI not found in sandbox home. Fetching installer..."
        apt-get install -y curl >/dev/null 2>&1
        curl -fsSL https://antigravity.google/cli/install.sh | bash
      fi
      
      # Ensure the binary is on the path and execute it with passed arguments
      export PATH="/root/.local/bin:$PATH"
      exec agy "$@"
    ' -- "$@"
}
```

#### Claude Code
Note: this command hasn't been verified yet, but was generated based on the Google Antigravity command above, so use at your own risk.
```
claude_sandboxed() {
  local target_dir="${1:-$(pwd)}"
  shift
  
  # 1. Create persistent config and binary folders on your Mac if they don't exist
  # write permissions need to be explicitly given to the config folder for token refresh
  mkdir -p "$HOME/.ai-sandbox-home/.local/bin"
  mkdir -p "$HOME/.ai-sandbox-home/.claude"
  chmod 700 "$HOME/.ai-sandbox-home/.claude"
  
  echo "Starting sandbox for directory: $target_dir"
  
  # 2. Run the container with an auto-installation fallback
  podman run --rm -it \
    -v "$HOME/.ai-sandbox-home:/root:Z" \
    -v "$target_dir:/workspace" \
    -w "/workspace" \
    debian:stable-slim bash -c '
      # always install certificates for authentication signing
      apt-get update && apt-get install -y ca-certificates >/dev/null 2>&1
      # Install claude to the mounted persistent volume if it is missing
      if [ ! -f /root/.local/bin/claude ]; then
        echo "Claude Code CLI not found in sandbox home. Fetching installer..."
        apt-get install -y curl nodejs npm >/dev/null 2>&1
        npm config set prefix /root/.local
        npm install -g @anthropic-ai/claude-code
      fi
      
      # Ensure the binary is on the path and execute it with passed arguments
      export PATH="/root/.local/bin:$PATH"
      exec claude "$@"
    ' -- "$@"
}
```

## Do This Each Time You Start Your Computer

### Podman
To start the VM, run:
```
podman machine start
```
And check that it has launched successfully by running:
```
podman info
```

### Running the Coding Assistant
Once the podman VM is running, use one of the custom commands defined in the previous section to launch a coding assistant within a container. If you are already in your working dir, you can just run one of the commands without any additional arguments e.g. for Google Antigravity:
```
agy_sandboxed
```
If you aren't in your working dir and want to mount it into the container, pass it as the first argument to the command e.g. for Google Antigravity:
```
agy_sandboxed ~/projects/myapp/
```
You can also pass additional arguments to the coding assistant, but these *must* be passed after the working dir e.g. for Google Antigravity with a specific model:
```
agy_sandboxed ~/projects/myapp/ --model="Gemini 3.5 Flash (Medium)"
```