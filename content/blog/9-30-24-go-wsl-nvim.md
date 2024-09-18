+++
author = "Cole Boren"
title = "Setting up, Running, and Debugging Go Applications with Neovim on WSL"
date = "2024-09-18"
description = "A first pass at my approach to setting up Neovim on WSL with support for Go, debugging, stepping through code, and streamlining my nvim setup process as a windows user."
tags = [
    "neovim",
    "configuration",
    "go",
    "wsl",
    "debugging"
]
+++

---

## Introduction

As a minimalist at heart, I've always been drawn to simplicity - that's part of what attracted me to Go. Years of skateboarding and wrist injuries, combined with my ADHD-induced tendency to get distracted by complex IDEs, led me to explore Vim motions and Neovim. While my ultimate goal is to run ArchLinux, blast some breakcore, and hack away on Neovim like a true power user, I'm not quite there yet. As a Windows user, I wanted to find a middle ground - a way to use Neovim and Go without completely abandoning my current setup. Enter WSL (Windows Subsystem for Linux). This guide is my first attempt at setting up a development environment that combines the best of both worlds: Neovim and Go on WSL, with the added ability to debug and step through code. Fair warning: this is just the beginning. Expect more blogs as I continue to refine and expand my Neovim setup. For now, let's dive into setting up WSL, Go, and the Debug Adapter Protocol (DAP) with Kickstart.nvim. We'll cover everything from WSL installation to configuring Neovim for Go development and debugging.

## Step 1: Installing WSL

1. Open PowerShell as Administrator and run:

   ```powershell
   wsl --install
   ```

2. Restart your computer when prompted.
3. After restart, open Ubuntu from the Start menu to complete the setup.

## Step 2: Installing Go

1. Update your package list:

   ```shell
   sudo apt update
   ```

2. Remove any existing Go installation:

   ```shell
   sudo rm -rf /usr/local/go
   ```

3. Download Go 1.22.7, you can use whatever go version you want here

   ```shell
   wget https://go.dev/dl/go1.22.7.linux-amd64.tar.gz
   ```

4. Extract Go to /usr/local:

   ```shell
   sudo tar -C /usr/local -xzf go1.22.7.linux-amd64.tar.gz
   ```

5. Set up your Go environment:

   ```shell
   echo 'export GOROOT=/usr/local/go' >> ~/.bashrc
   echo 'export PATH=$GOROOT/bin:$PATH' >> ~/.bashrc
   source ~/.bashrc
   ```

6. Verify the installation:

   ```shell
   go version
   ```

## Step 3: Installing Neovim

1. Download the latest Neovim AppImage:

   ```shell
   curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim.appimage
   ```

2. Make it executable:

   ```shell
   chmod u+x nvim.appimage
   ```

3. Move it to a directory in your PATH:

   ```shell
   sudo mv nvim.appimage /usr/local/bin/nvim
   ```

## Step 4: Setting Up Kickstart.nvim

1. Before we get started here,  be sure to checkout my [dotfiles](https://github.com/williycole/dotfiles) in case you want to see the final result or need a reference along the way. Also feel free to fork/clone them.
2. Back up your existing Neovim configuration:

   ```shell
   mv ~/.config/nvim ~/.config/nvim.bak
   ```

3. Clone Kickstart.nvim:

   ```shell
   git clone https://github.com/nvim-lua/kickstart.nvim.git ~/.config/nvim
   ```

## 5. Configuring Go Development and DAP

1. Installing Delve Debugger (moved up):

   ```shell
   go install github.com/go-delve/delve/cmd/dlv@latest
   ```

2. Instead of creating separate files(we can do that later), we'll modify the existing `init.lua` in the `custom/plugins` directory. Open `~/.config/nvim/lua/custom/plugins/init.lua` and add the following content:

```lua
   -- Function to find main.go file in the project incase its not in root
local function find_main_go()
  local root = vim.fn.getcwd()
  local main_go = vim.fn.globpath(root, '**/main.go', 0, 1)
  if #main_go > 0 then
    return vim.fn.fnamemodify(main_go[1], ':h')
  end
  return root
end

return {
  -- Core DAP (Debug Adapter Protocol) plugin
  {
    'mfussenegger/nvim-dap',
    dependencies = {
      -- Creates a beautiful debugger UI
      'rcarriga/nvim-dap-ui',

      -- Installs the debug adapters for you
      'williamboman/mason.nvim',
      'jay-babu/mason-nvim-dap.nvim',

      -- Add your own debuggers here
      'leoluz/nvim-dap-go',
    },
    config = function()
      local dap = require 'dap'
      local dapui = require 'dapui'

      -- Configure Mason to automatically install DAP adapters
      require('mason-nvim-dap').setup {
        -- Makes a best effort to setup the various debuggers with
        -- reasonable debug configurations
        automatic_setup = true,

        -- You can provide additional configuration to the handlers,
        -- see mason-nvim-dap README for more information
        handlers = {},

        -- You'll need to check that you have the required things installed
        -- online, please don't ask me how to install them :)
        ensure_installed = {
          -- Update this to ensure that you have the debuggers for the langs you want
          'delve',
        },
      }

      -- Basic debugging keymaps, feel free to change to your liking!
      vim.keymap.set('n', '<F5>', dap.continue, { desc = 'Debug: Start/Continue' })
      vim.keymap.set('n', '<F1>', dap.step_into, { desc = 'Debug: Step Into' })
      vim.keymap.set('n', '<F2>', dap.step_over, { desc = 'Debug: Step Over' })
      vim.keymap.set('n', '<F3>', dap.step_out, { desc = 'Debug: Step Out' })
      vim.keymap.set('n', '<leader>b', dap.toggle_breakpoint, { desc = 'Debug: Toggle Breakpoint' })
      vim.keymap.set('n', '<leader>B', function()
        dap.set_breakpoint(vim.fn.input 'Breakpoint condition: ')
      end, { desc = 'Debug: Set Breakpoint' })

      -- Dap UI setup
      -- For more information, see |:help nvim-dap-ui|
      dapui.setup {
        -- Set icons to characters that are more likely to work in every terminal.
        --    Feel free to remove or use ones that you like more! :)
        --    Don't feel like these are good choices.
        icons = { expanded = '▾', collapsed = '▸', current_frame = '*' },
        controls = {
          icons = {
            pause = '⏸',
            play = '▶',
            step_into = '⏎',
            step_over = '⏭',
            step_out = '⏮',
            step_back = 'b',
            run_last = '▶▶',
            terminate = '⏹',
            disconnect = '⏏',
          },
        },
      }

      -- Toggle to see last session result. Without this, you can't see session output in case of unhandled exception.
      vim.keymap.set('n', '<F7>', dapui.toggle, { desc = 'Debug: See last session result' })

      dap.listeners.after.event_initialized['dapui_config'] = dapui.open
      dap.listeners.before.event_terminated['dapui_config'] = dapui.close
      dap.listeners.before.event_exited['dapui_config'] = dapui.close

      -- Install golang specific config
      require('dap-go').setup()

      -- Override dap-go's launch configuration so we can find main.go even if it isn't in the root of our project
      dap.configurations.go = {
        {
          type = 'go',
          name = 'Debug',
          request = 'launch',
          program = function()
            return find_main_go()
          end,
        },
      }
    end,
  },
}

```

## 6. Final Configuration

1. Open Neovim: `nvim` and Run `:checkhealth` to ensure everything is set up correctly.
2. Optional tease apart the main `init.lua` file, ie. `nvim\init.lua`.

- To do this find different sections of code you would like to pull out of the main file. For example see steps and lua snippet below of the auto format configuration that comes with kickstart.
- **But before you do this be sure to uncomment  `-- { import = 'custom.plugins' },`** in `nvim\init.lua`, this will allow you to tease apart this main init.lua and place the things you tease apart in `nvim\custom\plugin\auto-format.lua`

```lua
-- Everything in the curly braces is alread in the nvim/init.lua, simply cut it out
-- and place it in a file within `nvim\custom\plugin\auto-format.lua`, and be sure
-- to include the return
-- ie. return { paste all the code you just cut here }
-- if you do this sytematically you can really clean up the main init.lua file at nvim root dir
return { -- Autoformat
  'stevearc/conform.nvim',
  event = { 'BufWritePre' },
  cmd = { 'ConformInfo' },
  keys = {
    {
      '<leader>f',
      function()
        require('conform').format { async = true, lsp_format = 'fallback' }
      end,
      mode = '',
      desc = '[F]ormat buffer',
    },
  },
  opts = {
    notify_on_error = false,
    format_on_save = function(bufnr)
      -- Disable "format_on_save lsp_fallback" for languages that don't
      -- have a well standardized coding style. You can add additional
      -- languages here or re-enable it for the disabled ones.
      local disable_filetypes = { c = true, cpp = true }
      local lsp_format_opt
      if disable_filetypes[vim.bo[bufnr].filetype] then
        lsp_format_opt = 'never'
      else
        lsp_format_opt = 'fallback'
      end
      return {
        timeout_ms = 500,
        lsp_format = lsp_format_opt,
      }
    end,
    formatters_by_ft = {
      lua = { 'stylua' },
      -- Conform can also run multiple formatters sequentially
      -- python = { "isort", "black" },
      --
      -- You can use 'stop_after_first' to run the first available formatter from the list
      -- javascript = { "prettierd", "prettier", stop_after_first = true },

      go = { 'go/fmt' }, -- NOTE: this isn't default, I added this for go formatting, the rest in this example is default
    },
  },
}

```

- Generally this approach looks roughly as follows.
  1. Plugin Location:
      - Plugins must be defined either in the main `nvim/init.lua` file or within the `lua/custom/plugins` directory.
  2. Modular Approach:
        - Gradually separating components from the main `init.lua` is an effective way to experiment with your setup.
        - Start by isolating small, manageable pieces of configuration.
  3. Hands-on Practice:
    This process also provides valuable Neovim editing practice:
        - Open the source file
        - Use `cc` to cut the desired code
        - Navigate to the target location with `:Ex`
        - Create a new file with `%` followed by the filename (e.g., `my-new-file.lua`)
        - Enter insert mode with `i`, press return, then `Esc`
        - Paste the code with `p`
  4. Troubleshooting:
        - When encountering plugin issues, always refer to the plugin's documentation and README.
        - Check for recent GitHub issues related to the plugin for potential solutions or known problems.

## Conclusion

Now you should be fully setup to Debug your Go projects with Neovim on WSL! Obviously there still a lot that could be done to further clean up this setup/debug experience, if you looked at or used my [dotfiles](https://github.com/williycole/dotfiles) you might have even noticed plenty of todos as I will likely iterate on this in the future and make another post as I learn/cleanup my neovim setup.

If you have any questions or trouble, feel free to leave a comment. I really hope this was helpful to someone, and I appreciate you if you made it this far and are reading this!
