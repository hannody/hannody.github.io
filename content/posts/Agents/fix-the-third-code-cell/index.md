---
title: "Fix the Third Code Cell"
date: 2026-08-15
draft: false
slug: "fix-the-third-code-cell"
tags:
  - vscode
  - jupyter
  - notebooks
  - ai-agents
  - developer-workflow
---

I was trying to get a coding agent to fix a few notebook cells, and it asked me to copy-paste snippets just so it could figure out which ones I meant. It worked, but it felt like a step backward. I knew exactly which cells I meant, but I had no direct way to point to them. Why does referencing a specific cell still require manual copy-pasting?

Usually, we point agents to notebook cells indirectly: *the cell that loads the dataset*, *the cell that does the groupby and filtering*, and so on. Some agents are smart enough to figure it out by inspecting the notebook, looking at the selected cell, or running a little Python to find the cell.

It works, but it is not a great way to address a cell. Insert a cell, delete one, or reorder the notebook, and positional references change. Descriptions can also be ambiguous when several cells look similar.

Then I noticed something interesting while watching the agent work: **it was already using the cell's `id` to find the cells I was talking about.**

The agent had a precise identifier for the cell. I did not.

That got me thinking: why can't we use the same language?

Instead of pasting an entire cell or describing which one I mean, I should be able to say:

> Fix cell `e66c99c5`. The groupby looks wrong.

Short, precise, and directly useful to both sides.

## The id that's already there

Since [nbformat 4.5](https://nbformat.readthedocs.io/en/latest/format_description.html#cell-ids), Jupyter notebook cells have an `id`:

```json
{
  "cell_type": "code",
  "id": "e66c99c5",
  "metadata": {},
  "source": [
    "df = load_dataset(path)\n",
    "df.head()"
  ]
}
```

The id gives the cell an identity independent of its position in the notebook.

So the information was already there. The agent could see it. I just could not easily see or copy it from VS Code.

That's what led me to build **Notebook Cell Id**.

## Notebook Cell Id

[Notebook Cell Id](https://marketplace.visualstudio.com/items?itemName=Mohanad-Abu-Nayla.notebook-cell-id) is a small VS Code extension that shows each notebook cell's nbformat id directly in the cell's status bar.

![A notebook in VS Code with each cell's nbformat id shown in its status bar; the badge of one cell is highlighted](screenshot.png)

Each cell gets a small badge such as:

`⧉ #5 e66c99c5`

Click it and the id is copied.

There are also two Command Palette commands:

* **Notebook: Copy Cell Id**
* **Notebook: Copy Cell Id with Notebook Path**

The second one gives you something like:

```text
notebooks/step_3.ipynb cell e66c99c5
```

This is useful when the agent is working across multiple notebooks.

The index is optional, and the status bar item can be disabled if you do not want it visible. The commands remain available.

## How it works

The extension is deliberately simple.

VS Code's built-in notebook support exposes the nbformat cell id through `NotebookCell.metadata.id`, so the extension does not parse `.ipynb` files or maintain its own mapping.

It simply uses `NotebookCellStatusBarItemProvider` to display the id belonging to each cell.

Move a cell, add one, delete one, and the reference follows the cell. VS Code's notebook serializer preserves the id when the notebook is saved.

The entire extension is roughly a hundred lines of plain JavaScript with no build step. The source is available on [GitHub](https://github.com/hannody/vscode-notebook-cell-id) under the MIT license.

One caveat: cells only carry ids in **nbformat 4.5 and newer**. Older notebooks show no badges — but re-saving them in any current Jupyter or VS Code adds ids to every cell, so the fix is one save away.

[Install Notebook Cell Id](https://marketplace.visualstudio.com/items?itemName=Mohanad-Abu-Nayla.notebook-cell-id)

Sometimes the information you need is already there.

The agent can see it.

I just wanted to see it too.
