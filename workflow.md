# A Claude Code Workflow for HPC Environments 

As an ML researcher, I spend a considerable amount of time working through messy GitHub repos - some by myself, others by others. Clean, minimal, working code on ML repos is hard to find, particularly those that are considered supplementary material for research papers at the top conferences in our field. While this fundamental problem is real, and we would all like it to be solved first, for all practical purposes, it's probably better to efficiently work around it.

In this document, I will outline how I use **Claude Code** - Anthropic's flagship terminal development environment - to automate a lot of the boring stuff that are bugs (not features) of the ML researcher's life. The pipeline is very simple. We are given:

- A GitHub repo that we are interested in, and
- A SLURM environment on a GPU cluster.

We want to get things running as fast as possible to actually do the work we want to do. Specifically, we want to navigate the repo, build a map, install dependencies, run SLURM jobs, and monitor them.

The config is available in [this](https://github.com/rwchakra/claude-code-hpc) GitHub repo, and this document serves as a reading guide to those files.


## Step 1: `CLAUDE.md`

Home base. Here, I write down the details of the SLURM environment I am working in - in the Nordics, it's usually LUMI, or a country's own GPU cluster, such as Norway's Olivia. These details are important because every step that exists downstream of this - installing dependencies, running jobs, and monitoring them - needs knowledge of the environment the code operates in.

Therefore, in this file, I write about:

- The GPUs available (AMD MI250s on LUMI, NVIDIA GH200s on Olivia)
- The architectures on the compute and login nodes (these may well be different, e.g. x86 and ARM)
- Installation instructions (`conda`/`pip` installations on login nodes are strictly frowned upon)
- Cache paths for torchhub/huggingface
- A general SLURM script template

## Step 2: Navigating a Repo

I understand that gitingest and DeepWiki exist, and they are quite useful, but I found it much smoother for me to keep everything in one context, and not have to switch from my terminal at all. I write about the usefulness of Claude Code (`cc`) with respect to this aspect in a bit more detail later.

After a `git clone`, I wish to *not* have to go through the directories, `README.md`, and the core files to get everything up and running manually. I could ask Claude instead to build a **repo map** - a structured understanding of the most important units of the repo and how they interact.

I do this as a skill called, quite creatively, `navigate-repo.md`. I put this inside `.claude/skills/navigate-repo/SKILL.md`. This structure is so that I can invoke this skill as a slash command on the Claude terminal, like `/navigate-repo`, which Claude can automatically find. Please see [here](https://docs.anthropic.com) for more details on skills and slash commands - it's worth reading this.

Then, on the terminal, one could just write `/navigate-repo` or `Reproduce <the repo you cloned>, through /navigate-repo` on the `cc` terminal, and thus starts a multi-turn conversation that will eventually work out. I do not `-dangerously-skip-permissions`, and do follow `cc`'s process quite closely (`Ctrl+O` is your friend here). More on this later.

In the end, I ask `cc` to create (again, imaginatively named) `REPO_MAP.md`, which can be referenced by Claude on any session in the future.


## Step 3: Resolving Dependencies

This is the biggest pain point for me. It is likely that the repo the paper links to, while useful, is also dependency hell - wheels may not be available depending on the computer you are on, library versions may be incompatible, among other things. The `issues` tab sometimes highlights some of these, but you're pretty much on your own. Depending on how clean the repo is, you're looking at 1-2 days of debugging.

**I used `cc` to do this in 20-25 minutes.**

What you need, again, is a good skill file. In this case, I named it `resolve-deps.md`. This contains some well-known rules for installing things on LUMI (checking for CUDA-specific dependencies is important since AMD uses `rocm`) and Olivia. These rules are mostly taken from the official docs, which is always useful to skim through and add to the skill file.

Another thing that needs to go in here is where the pre-built `.sif` containers are available. Installing packages on top of `.sif` is the fastest way to do things, I imagine - but I am not well versed in this issue. If someone out there has some insights on this, please let me know and off it will go to `resolve-deps.md`.

This file lives in `.claude/skills/resolve-deps/SKILL.md`. Therefore, after the repo map is done, we can do `/resolve-deps and install requirements` on the terminal. Here, Claude goes to work - working through issues, asking for certain permissions, debugging, fixing, `<verbing-claude-verbs>`. In 5-10 minutes, I have a full working environment. This wasn't entirely unsupervised or autonomous-I had to confirm some suggestions, reject others - but the pain point is gone.

I ask Claude to save the summary of the environment and gotchas to `ENV.md`.

## Step 4: Sample Run

Assuming you have the datasets downloaded (this could also be a Claude skill, by the way), I want a simple run where the model and data are loaded onto the GPU, and one forward pass is executed. We want to check whether things break and how. I also want a clean summary of the dataset and feature shapes, and output logs.

The great thing is that, if things break (and they will), `cc` will work through those issues itself, saving you the effort to go on those infinite GitHub issue threads and hoping for a match. The reality is that most issues aren't novel, and Opus's training set has probably ingested all known training issues in the history of PyTorch.

If at some point Opus indeed loses itself in its verbs, and I do find something novel, it will be added to `ENV.md`. This is the power of this framework. It's instant, zero-friction context that is persistent.

## Step 5: Monitor SLURM

Okay, we have an understanding of the repo, a working env, and our forward pass works. For longer jobs, we need one additional thing - a way to monitor active jobs.

In `./claude/skills/monitor-slurm/SKILL.md`, we add the instructions to do that: `squeue`, `<path_to_logs>`, checking for `.err` and `.out`, summarizing issues, and working through them. Again, accessible through a simple `/monitor-slurm` on the terminal.

It doesn't always have to "fix" issues. In fact, even for jobs that are running, it will give you throughput, ETA, progress, etc. All you have to do is ask. Everything else is just context.


## The Workflow

There are certain other tweaks I do to improve context across sessions. One is `SCRATCH.md`. This is a diary of sorts - a big picture summary of updates for the day. This includes ideas we have been playing with, implementational changes, training and inference times, etc.

Every day (or every session, even), I write `@SCRATCH.md` in the terminal to load it into `cc`'s context, and chat with it - what are we thinking? What's the chronology of events? Are we progressing? And so on.

I understand that this design may actually not be the best way to do things - there's always a higher level of abstraction, but I haven't yet worked out what that looks like, or in the event I do, what the opportunity cost is of moving there. In time.

The goal with this file is that it will grow into a full-fledged story of how research progresses for a particular paper. I am curious myself how that evolution looks. If it works, we could think about having the whole community upload their custom-made `SCRATCH.md` to some forum, for everyone to visualize how research evolves across the world. A git for ideas.

## A note on token usage
It's also useful to keep a log of token burn - unfortunately, none of this is free (yet). But honestly, *not* paying for Opus 4.6 just seems illegal. Here's a log of a usual session on my current project:

 - Used Haiku to create a simple `SKILL.md` file.
 - Used Opus max-effort to fetch `SCRATCH.md` in context, and outline the steps for the day
 - Remaining part of the session was entirely getting the feature extraction scripts to run and debugging them. Minimal debugging.
 - Claude was running srun, automatically monitoring the logs, fixing the errors on its own, and running again. With permissions.
 - Whole session was less than a couple of hours - most of that time was spent waiting for the scripts to finish.
 - Total tokens consumed - 142,000 messages + 22,000 system = 164,000.
 - About 42% of the session limit on the base Pro Plan - 10% of the weekly limit.

Whether the above seems expensive or cheap to you depends entirely on what you do. Rule of thumb - simple fixes, go for Haiku, complex fixes, go for Opus. I do not think Sonnet has any role to play in future releases, but that's just me. If you have figured out some sort of infinite token glitch, use Opus 4.6 on High all the time.  

## Closing Thoughts

Claude Code has completely changed *one particular* aspect of my work for the better - getting things to run without wasting brain cycles on debugging issues that will teach me nothing. In this way, I can just get on with the things that I actually need to do - read papers, learn new concepts, develop methodologies, write papers, and most importantly, *think*.

As far as execution is concerned, particularly for the small-scale environments that academia usually operates in, I believe this is, by and large, solved. And that's a *good* thing. Finally, we can just do what we feel like doing - things that we actually enjoy.

Which is why I do not let Claude anywhere near my paper writing (except when I need to add a citation to `.bib`, or change some color on `tikz`), or developing a novel method/analysis, or reading and summarizing papers. I do not like AI summaries of papers, and AI-written papers either - but I am aware that this is because those are the things I actually enjoy doing: reading and writing. Others may love programming and debugging way more than writing, and they could modify `.claude` accordingly.

I have not yet bought into the idea of full-fledged autonomous research agents. This misplaces what research aims to be, and more importantly, who research is *for*. The invariant, however, is the nature of interaction - work with the model, not against it. Take turns, have a conversation. Figure it out together. Zero context switching. `tmux` your way through it. One pane has `cc`, the other has a file viewer (`yazi`, maybe?), another has the code you are looking at. Or a PDF. Or a spreadsheet. Whatever.

In a weird way, I think Claude Code will solve the attention crisis of the modern economy, since everything you could possibly achieve is on one window exclusively - not on different apps on different windows.

What is common for everyone is that after about five decades of FOSS, we have come back full circle - a terminal really is all you need. This time, in plain English.
