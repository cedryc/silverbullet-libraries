---
name: Library/zefhemel/Git
tags: meta/library
---
This library adds a basic git synchronization functionality to SilverBullet. It should be considered a successor to [silverbullet-git](https://github.com/silverbulletmd/silverbullet-git) implemented in Space Lua.

The following commands are currently implemented:

${widgets.commandButton("Git: Status")}

- Shows `git status` in a side panel with close and refresh buttons

${widgets.commandButton("Git: Add Page")}

- Stages (`git add`) the file for the page you're currently editing

${widgets.commandButton("Git: Commit")}

- Shows a panel listing currently staged files
- Lets you type a commit message
- Commits only the staged files (equivalent to `git commit`)

${widgets.commandButton("Git: Commit All")}

- Shows a panel listing all pending files (staged, unstaged, and untracked)
- Lets you type a commit message
- Stages everything and commits (equivalent to `git add -A && git commit`)

${widgets.commandButton("Git: Sync")}

* Runs `Git: Commit All` with the default "Snapshot" commit message
* `git pull`s changes from the remote server
* `git push`es changes to the remote server


> **Note:** `Git: Commit` previously ran `git commit -a` (committing
> all changes, not just staged ones). It now only commits staged
> files. Use `Git: Commit All` for the previous behavior.


# Configuration
There is currently only a single configuration option: `git.autoSync`. When set, the `Git: Sync` command will be run every _x_ minutes.

Example configuration:
```lua
config.set("git.autoSync", 5)
```

# Implementation
The full implementation of this integration follows.

## Configuration
```space-lua
-- priority: 100
config.define("git", {
  type = "object",
  properties = {
    autoSync = schema.number()
  }
})
```

## Commands
```space-lua
git = {}

function checkedShellRun(cmd, args)
  local r = shell.run(cmd, args)
  if r.code != 0 then
    error("Error: " .. r.stdout .. r.stderr, "error")
  end
end

function git.porcelainStatus()
    local result = shell.run("git", {"status", "--porcelain"})
    local entries = {}
    for line in (result.stdout .. "\n"):gmatch("(.-)\n") do
        if line:match("%S") then
            table.insert(entries, {
                indexStatus = line:sub(1, 1),
                path = line:sub(4)
            })
        end
    end
    return entries
end

function git.pendingFiles()
    local files = {}
    for _, e in ipairs(git.porcelainStatus()) do
        table.insert(files, e.path)
    end
    return files
end

function git.stagedFiles()
    local files = {}
    for _, e in ipairs(git.porcelainStatus()) do
        if e.indexStatus ~= " " and e.indexStatus ~= "?" then
            table.insert(files, e.path)
        end
    end
    return files
end

function git.localChanges()
    return #git.pendingFiles() > 0
end

-- Check the state of statusPanel
git.statusPanelOpen = git.statusPanelOpen or false

function git.status()
    local result = shell.run("git", {"status"})
    if result.code ~= 0 then
        editor.flashNotification("Git status error: " .. result.stderr, "error")
        return
    end

    git.statusPanelOpen = true

    local refreshBtn = dom.button {
        title = "Refresh",
        onclick = function() git.status() end,
        "↻"
    }
    local closeBtn = dom.button {
        title = "Close",
        onclick = function()
            editor.hidePanel("rhs")
            git.statusPanelOpen = false
        end,
        "✕"
    }
    local scrollArea = dom.div {
        style = "height: calc(100vh - 3rem); overflow-y: auto;",
        dom.pre { result.stdout }
    }
    local content = dom.div {
        style = "height: 100%; display: flex; flex-direction: column;",
        dom.div { style = "display: flex; justify-content: flex-end; gap: 0.3rem;", refreshBtn, closeBtn },
        scrollArea
    }

    editor.showPanel("rhs", 0.3, content)
end

function git.refreshStatusIfOpen()
    if git.statusPanelOpen then
        git.status()
    end
end

function git.addCurrentPage()
    local page = editor.getCurrentPage()
    local path = page .. ".md"

    local ok, err = pcall(function()
        checkedShellRun("git", {"add", path})
    end)

    if not ok then
        editor.flashNotification("Erreur git add : " .. err, "error")
        return
    end

    git.refreshStatusIfOpen()
    editor.flashNotification("Added to staging : " .. path)
end


function git.commit(message, fileList)
    if not message or message == "" then
        message = "Updated " .. (fileList or "")
    end

    local ok, err = pcall(function()
        checkedShellRun("git", {"commit", "-m", message})
    end)

    git.refreshStatusIfOpen()

    if ok then
        editor.flashNotification("committed " .. (fileList or ""), "info")
    else
        editor.flashNotification("Git commit failed: " .. err, "error")
    end
end

function git.commitAll(message, fileList)
    if not git.localChanges() then
        editor.flashNotification("Nothing to commit", "info")
        return
    end

    if not message or message == "" then
        message = "Updated " .. (fileList or "")
    end

    local ok, err = pcall(function()
        checkedShellRun("git", {"add", "-A"})
        checkedShellRun("git", {"commit", "-m", message})
    end)

    git.refreshStatusIfOpen()

    if ok then
        editor.flashNotification("committed " .. (fileList or ""), "info")
    else
        editor.flashNotification("Git commit failed: " .. err, "error")
    end
end

function git.showCommitPanel(files, commitFn)
    if #files == 0 then
        editor.flashNotification("Nothing to commit", "info")
        return
    end

    local typedMessage = ""

    local fileItems = {}
    for _, f in ipairs(files) do
        table.insert(fileItems, dom.div { style = "font-family: monospace; font-size: 0.85em; padding: 0.1rem 0;", f })
    end

    local fileListBox = dom.div {
        style = "max-height: 40vh; overflow-y: auto; margin-bottom: 0.5rem; border-bottom: 1px solid #666; padding-bottom: 0.5rem;",
        table.unpack(fileItems)
    }

    local input = dom.input {
        type = "text",
        placeholder = "Commit message",
        style = "width: 100%; box-sizing: border-box; margin-bottom: 0.5rem;",
        oninput = function(event)
            typedMessage = event.target.value
        end
    }

    local confirmBtn = dom.button {
        onclick = function()
            editor.hidePanel("rhs")
            commitFn(typedMessage)
        end,
        "Commit"
    }

    local cancelBtn = dom.button {
        onclick = function()
            editor.hidePanel("rhs")
        end,
        "Cancel"
    }

    local content = dom.div {
        style = "padding: 0.5rem;",
        dom.div { style = "font-weight: bold; margin-bottom: 0.5rem;", "Files to commit:" },
        fileListBox,
        input,
        dom.div { style = "display: flex; justify-content: flex-end; gap: 0.3rem;", cancelBtn, confirmBtn }
    }

    editor.showPanel("rhs", 0.4, content)
end

function git.sync()
    local files = git.pendingFiles()
    git.commitAll(nil, table.concat(files, ", "))

    local ok, err = pcall(function()
        print "Pulling..."
        checkedShellRun("git", {"pull"})
        print "Pushing..."
        checkedShellRun("git", {"push"})
    end)

    git.refreshStatusIfOpen()

    if ok then
        editor.flashNotification("Sync done!")
    else
        editor.flashNotification("Sync failed: " .. err, "error")
    end
end

command.define {
    name = "Git: Status",
    run = git.status
}

command.define {
    name = "Git: Add Page",
    run = git.addCurrentPage
}

command.define {
    name = "Git: Commit",
    run = function()
        local files = git.stagedFiles()
        local fileListStr = table.concat(files, ", ")

        git.showCommitPanel(files, function(msg)
            git.commit(msg, fileListStr)
        end)
    end
}

command.define {
    name = "Git: Commit All",
    run = function()
        local files = git.pendingFiles()
        local fileListStr = table.concat(files, ", ")

        git.showCommitPanel(files, function(msg)
            git.commitAll(msg, fileListStr)
        end)
    end
}

command.define {
  name = "Git: Sync",
  run = git.sync()
}

```

```space-lua
-- priority: -1
local autoSync = config.get("git.autoSync")
if autoSync then
  print("Enabling git auto sync every " .. autoSync .. " minutes")

  local lastSync = os.time()

  event.listen {
    name = "cron:secondPassed",
    run = function()
      local now = os.time()
      if (now - lastSync)/60 >= autoSync then
        lastSync = now
        git.sync()
      end
    end
  }
end

```
