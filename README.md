```
                            ▄
█▀▀▀ █▀▀▄ █▀▀█ █▀▀▀ █▀▀█ █▀▀█ █▀▀█
▀▀▀█ █  █ █  █ █    █  █ █  █ █▀▀▀
▀▀▀▀ ▀  ▀ ▀▀▀▀ ▀▀▀▀ ▀▀▀▀ ▀▀▀▀ ▀▀▀▀
```

snocode is currently in a very early stage of development, and it hasn't yet reached a usable state.

snocode is a minimal AI agent built for the terminal, written in Rust. It aims to take inspiration from existing agents, such as aider, opencode, and codex, while rethinking common elements of the TUI, keeping essential features, and stripping out anything not strictly needed.

It has a few key differences from existing coding agents, some of which are listed here:
 - The agent never writes commits to the repository. This helps you keep its changes in check and avoids git annoyances.
 - The app automatically condenses each of the model's outputs to take up a fixed maximum amount of the screen, allowing you to view several past messages at a time. Instead of showing the huge paragraphs the model spits out while running tests or implementing a feature, most output text is condensed by default, showing only a "report" which is limited to a configurable number of characters (with everything else viewable on demand).
 - "Ask" (read-only) mode is clearly separated from "work" (read-write) mode. In ask mode, models can output more text or create plans as reviewable artifacts rather than just outputting a wall of text once.
 - Models can directly cite outputs of previous commands as proof something really happened. This keeps the model's claims easy to check against reality. 

snocode is licensed under the [GPL (version 3.0 only)](LICENSE).
