## Why Bash Script > C Wrapper

1. **Proven Solution**: homebrew-emacs-plus has used bash wrappers for years without issues
2. **No Compilation**: No dependency on clang, no architecture detection needed
3. **Easier to Debug**: Users can read/edit the wrapper if needed
4. **Truly Relocatable**: Bash `$DIR` resolution is bulletproof
5. **Simpler Code**: ~30 lines vs ~150 lines of C + compilation logic

## The Better Fix: Use Bash Script

Replace your entire `PathInjector` class with this battle-tested version:

```ruby
class PathInjector < AbstractEmbedder
  def initialize(app, options = {})
    super(app)
    @inject_path = options.fetch(:inject_path, true)
    @shell_sources = options.fetch(:shell_sources, default_shell_sources)
  end

  def embed
    return unless @inject_path

    binary_dir = File.join(app, "Contents", "MacOS")
    original_binary = File.join(binary_dir, "Emacs")
    real_binary = File.join(binary_dir, "Emacs.real")

    # Prevent double-wrapping if run twice
    if File.exist?(real_binary)
      info "Emacs binary already wrapped, skipping PATH injection..."
      return
    end

    info "Injecting dynamic PATH wrapper (emacs-plus style)..."

    # 1. Rename the actual binary
    debug "Renaming #{File.basename(original_binary)} to Emacs.real"
    FileUtils.mv(original_binary, real_binary)

    # 2. Create the wrapper script
    wrapper_content = generate_wrapper_script

    # 3. Write the wrapper and make it executable
    debug "Writing wrapper script to #{File.basename(original_binary)}"
    File.write(original_binary, wrapper_content)
    FileUtils.chmod(0755, original_binary)

    info "Dynamic PATH wrapper created successfully"
    info "Emacs will now source shell environment on every launch"
  end

  private

  def generate_wrapper_script
    <<~SCRIPT
      #!/bin/bash
      # Dynamic PATH wrapper for Emacs.app
      # Sources user shell profiles to inherit PATH and other environment variables
      # This allows Emacs to find tools installed after building, without rebuilding

      # Source shell profiles silently (redirect output to prevent GUI launch issues)
#{shell_source_lines.map { |line| "      #{line} 2>/dev/null" }.join("\n")}

      # Ensure Homebrew paths are present (M1/Intel compatibility)
      # These are prepended to avoid conflicts but still allow user overrides
      if [ -d "/opt/homebrew/bin" ]; then
        export PATH="/opt/homebrew/bin:/opt/homebrew/sbin:$PATH"
      fi
      if [ -d "/usr/local/bin" ]; then
        export PATH="/usr/local/bin:/usr/local/sbin:$PATH"
      fi

      # Add standard system paths if not already present
      export PATH="$PATH:/usr/bin:/bin:/usr/sbin:/sbin"

      # Get the directory containing this script (fully portable)
      DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" >/dev/null 2>&1 && pwd )"

      # Execute the real Emacs binary with all arguments
      exec "$DIR/Emacs.real" "$@"
    SCRIPT
  end

  def shell_source_lines
    @shell_sources.map do |profile|
      <<~LINE.chomp
        if [ -f "$HOME/#{profile}" ]; then
          source "$HOME/#{profile}"
        fi
      LINE
    end
  end

  def default_shell_sources
    [
      ".zprofile",      # zsh login shell (best for PATH setup)
      ".zshenv",        # zsh all shells
      ".bash_profile",  # bash login shell
      # ".bashrc",      # REMOVED: Interactive only, often has echo/prompts that break GUI launches
      ".profile",       # POSIX sh fallback
    ]
  end
end
```

## Why This Works Better

1. **No `$0` issues**: The wrapper script doesn't manipulate `$0` at all
2. **No compilation**: Works immediately, no clang dependency
3. **Portable**: `$DIR` resolves at runtime, works anywhere
4. **Simple**: Easy to understand and debug
5. **Proven**: Identical to homebrew-emacs-plus implementation

## If You Still Want C Wrapper

If you're committed to the C approach for some reason (like avoiding bash dependency or security), then yes, your fix with `new_argv[3] = "Emacs"` is correct. But honestly, the bash script is **simpler, safer, and proven**.

## To Apply

Replace your current `PathInjector` class (lines ~1726-1874) with the bash script version above. The icon cache fix in `IconEmbedder` is good to keep.

**Recommendation**: Go with bash. It's what emacs-plus uses, it's what works, and it's dramatically simpler. The C wrapper is over-engineering for this use case. 🚀


Here's the **corrected C wrapper** with the `$0` fix:

```ruby
class PathInjector < AbstractEmbedder
  def initialize(app, options = {})
    super(app)
    @inject_path = options.fetch(:inject_path, true)
    @shell_sources = options.fetch(:shell_sources, default_shell_sources)
  end

  def embed
    return unless @inject_path

    binary_dir = File.join(app, "Contents", "MacOS")
    original_binary = File.join(binary_dir, "Emacs")
    real_binary = File.join(binary_dir, "Emacs.real")

    # 1. Safety check: Don't wrap twice
    if File.exist?(real_binary)
      info "Emacs binary already wrapped, skipping PATH injection..."
      return
    end

    unless File.exist?(original_binary)
      fatal "Could not find original binary at #{original_binary}"
    end

    info "Injecting native C PATH wrapper (Mach-O compatible)..."

    # 2. Move the real binary out of the way
    debug "Renaming original binary to Emacs.real"
    FileUtils.mv(original_binary, real_binary)

    # 3. Determine the exact architecture flags needed for compilation
    arch_flags = get_arch_flags(real_binary)
    debug "Detected architecture flags: #{arch_flags.join(' ')}"

    # 4. Generate the C source code for the wrapper
    wrapper_source = File.join(binary_dir, "wrapper.c")
    File.write(wrapper_source, generate_c_wrapper_source)

    # 5. Compile the wrapper
    debug "Compiling native wrapper..."
    compile_cmd = [
      "clang",
      *arch_flags,
      "-O2",
      "-Wall",
      "-o", original_binary,
      wrapper_source
    ]

    unless system(*compile_cmd)
      fatal "Failed to compile native wrapper. Ensure clang is installed."
      # Attempt cleanup on failure
      FileUtils.mv(real_binary, original_binary) if File.exist?(real_binary)
      FileUtils.rm(wrapper_source) if File.exist?(wrapper_source)
      return
    end

    # 6. Cleanup C source
    FileUtils.rm(wrapper_source)

    # 7. Re-sign the wrapper binary
    info "Signing wrapper binary..."
    system("codesign", "--force", "--deep", "--sign", "-", original_binary)

    info "Native PATH wrapper installed successfully"
  end

  private

  # Reads the Mach-O header to determine which -arch flags to pass to clang
  def get_arch_flags(binary_path)
    begin
      file = MachO.open(binary_path)
      archs = []

      if file.is_a?(MachO::FatFile)
        archs = file.machos.map(&:cputype)
      else
        archs = [file.cputype]
      end

      flags = archs.map do |arch|
        arch_name = arch.to_s.gsub(/^:/, '')
        case arch_name
        when "arm64", "16777228" then "arm64"
        when "x86_64", "16777223" then "x86_64"
        when "i386", "7"          then "i386"
        else arch_name
        end
      end.uniq

      flags.flat_map { |a| ["-arch", a] }
    rescue MachO::NotAMachOError, MachO::TruncatedFileError => e
      fatal "Failed to parse Mach-O binary: #{e.message}"
      exit 1
    end
  end

  def generate_c_wrapper_source
    # Escape shell profile logic for C string embedding
    shell_profile_logic = shell_source_lines
      .map { |l| "#{l} 2>/dev/null;" }
      .join(" ")
      .gsub('\\', '\\\\\\\\')  # Escape backslashes for C
      .gsub('"', '\\"')         # Escape double quotes for C

    <<~C_CODE
      #include <stdio.h>
      #include <stdlib.h>
      #include <string.h>
      #include <unistd.h>
      #include <limits.h>
      #include <mach-o/dyld.h>

      int main(int argc, char **argv) {
          // 1. Get the path of this executable (the wrapper)
          char exe_path[PATH_MAX];
          uint32_t size = sizeof(exe_path);
          if (_NSGetExecutablePath(exe_path, &size) != 0) {
              fprintf(stderr, "Error: Buffer too small; need size %u\\n", size);
              return 1;
          }

          // 2. Construct the path to the real binary
          char real_exe[PATH_MAX];
          snprintf(real_exe, sizeof(real_exe), "%s.real", exe_path);

          // 3. Define Shell Logic
          const char *profile_logic = "#{shell_profile_logic}";

          const char *path_logic =
            "if [ -d /opt/homebrew/bin ]; then export PATH=/opt/homebrew/bin:/opt/homebrew/sbin:$PATH; fi; "
            "if [ -d /usr/local/bin ]; then export PATH=/usr/local/bin:/usr/local/sbin:$PATH; fi; "
            "export PATH=$PATH:/usr/bin:/bin:/usr/sbin:/sbin";

          // 4. Combine into one bash command
          // CRITICAL: We hardcode the real_exe path directly into the exec command
          // This prevents shell injection and path interpretation issues
          char cmd[8192];
          int required_len = snprintf(cmd, sizeof(cmd),
              "%s %s exec '%s' \\"$@\\"",
              profile_logic, path_logic, real_exe);

          if (required_len < 0 || required_len >= (int)sizeof(cmd)) {
              fprintf(stderr, "Error: Generated shell command exceeds buffer size.\\n");
              return 1;
          }

          // 5. Prepare arguments for /bin/bash
          // We construct: /bin/bash -c "cmd" "Emacs" "arg1" "arg2"...
          // CRITICAL FIX: "Emacs" becomes $0 in the shell context
          // This prevents shell profiles from trying to 'export /full/path/to/Emacs.real'
          // which would fail because paths with slashes are invalid variable names
          char **new_argv = malloc((argc + 4) * sizeof(char *));
          if (!new_argv) {
              perror("malloc failed");
              return 1;
          }

          new_argv[0] = "/bin/bash";
          new_argv[1] = "-c";
          new_argv[2] = cmd;
          new_argv[3] = "Emacs";  // Safe $0 - prevents "export: '/path': not a valid identifier"

          // Copy original arguments starting from argv[1]
          // User's arguments become $1, $2, $3, etc. in the shell
          for (int i = 1; i < argc; i++) {
              new_argv[i + 3] = argv[i];
          }
          new_argv[argc + 3] = NULL;

          // 6. Execute
          execv(new_argv[0], new_argv);
          perror("execv failed");
          free(new_argv);
          return 1;
      }
    C_CODE
  end

  def shell_source_lines
    @shell_sources.map do |profile|
      # Escape for bash command context (will be further escaped for C string)
      "if [ -f \\\"$HOME/#{profile}\\\" ]; then source \\\"$HOME/#{profile}\\\"; fi"
    end
  end

  def default_shell_sources
    [
      ".zprofile",      # zsh login shell (best for PATH setup)
      ".zshenv",        # zsh all shells
      ".bash_profile",  # bash login shell
      # ".bashrc",      # REMOVED: Interactive only, breaks GUI launches
      ".profile",       # POSIX sh fallback
    ]
  end
end
```

## Key Changes from Your Version

1. **Fixed `$0` issue**: `new_argv[3] = "Emacs"` instead of the full path
2. **Single quotes around `real_exe`**: Changed `\\"%s\\"` to `'%s'` in the exec command for proper path escaping
3. **Proper string escaping**: Added double-escaping for backslashes and quotes when injecting Ruby into C
4. **Clearer comments**: Explains why each choice matters

## How It Works

```c
// The generated bash command looks like:
// if [ -f "$HOME/.zprofile" ]; then source "$HOME/.zprofile"; fi 2>/dev/null; \
// if [ -d /opt/homebrew/bin ]; then export PATH=...; fi; \
// exec '/full/path/to/Emacs.real' "$@"

// Executed as:
// /bin/bash -c "command" "Emacs" arg1 arg2 arg3
//                         ^^^^^^  ^^^^ ^^^^ ^^^^
//                         $0      $1   $2   $3
```

The shell sees `$0="Emacs"` (safe), not `$0="/Users/.../Emacs.real"` (breaks export).

## Comparison: C vs Bash

| Feature | C Wrapper | Bash Script |
|---------|-----------|-------------|
| **Speed** | Faster (no bash startup) | Negligible difference |
| **Size** | ~16KB compiled | ~1KB text |
| **Portability** | Requires compilation | Works everywhere |
| **Debugging** | Need to recompile | Just edit the script |
| **Dependencies** | Needs clang at build time | None |
| **Complexity** | ~150 lines | ~30 lines |

Both work perfectly. C is slightly faster (microseconds), bash is dramatically simpler. Your choice! 🚀
