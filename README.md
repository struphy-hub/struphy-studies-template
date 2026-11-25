# Struphy studies template

A template for **private** collections of numerical studies with [Struphy](https://github.com/struphy-hub/struphy),
shared with chosen collaborators and published later if you want to. It has the same tools and the same study format
as the public site [struphy-studies](https://github.com/struphy-hub/struphy-studies)
([site](https://struphy-hub.github.io/struphy-studies/)), so a study can be moved there unchanged.

Each study has a folder `studies/<yyyy-mm>-<name>/` with
- `study.yaml`: title, date, Struphy commit, descriptions, main findings, and the runs with their commands;
- `params/`: the parameter and post-processing files of the runs (and further scripts of the study);
- `thumbnail.png`: the picture on the landing page.

The figures and videos are **not in git**. They live in a folder on [MPCDF DataShare](https://datashare.mpcdf.mpg.de),
shared with a public link (without password), and the pages embed them from there.

## What is private

| | where | who can see it |
|---|---|---|
| `study.yaml`, `params/`, thumbnails | your private repository | its members |
| the site (HTML) | an artifact of each CI run, or built locally | its members (and whoever you send the zip to) |
| figures and videos | DataShare folder with a public link | **anyone who has the link** (it is not listed anywhere, but it is not protected) |

So only put media on DataShare that may leak without harm; keep everything else in the repository. GitHub Pages
stays off: on most GitHub plans the Pages of a private repository are public, and the workflow never deploys from a
private repository.

## Set up a copy (once)

1. Create a private repository from this template: on GitHub **Use this template > Create a new repository**, choose
   the owner and **Private**. Or with the GitHub CLI:
   ```bash
   gh repo create <owner>/<name> --private --template struphy-hub/struphy-studies-template --clone
   ```
   (A *fork* of a public repository cannot be private; the template is the way to get a private copy.)
2. Add the collaborators: **Settings > Collaborators** (personal account) or **Collaborators and teams**
   (organization), with *Read* (view and download the site) or *Write* (add studies).
3. Edit `mkdocs.yml`: `site_name` (the title), `repo_url: https://github.com/<owner>/<name>` and `repo_name`.
   The study pages link to the study folders in `repo_url`, and CI fails while it points elsewhere.
   Edit `intro.md`, the text of the landing page.
4. Commit and push. The workflow **Site** builds the site (see below).

## Add a study

```bash
python -m venv .venv && .venv/bin/pip install -r requirements.txt   # once
source .venv/bin/activate

# 1. study folder and media folder from the Struphy output folders of the runs
python tools/study.py create 2026-11-my-study --title "My study" \
    --run base=/path/to/sim_base --run fine=/path/to/sim_fine \
    --figure /path/to/comparison.png --thumbnail /path/to/picture.png
```

`create` writes `studies/<slug>/study.yaml` (with TODOs), copies the params files, makes the thumbnail and puts the
figures and videos of the runs (from `<run>/results/`) into `~/struphy-studies-media/<slug>/`. Then:

1. Fill in the TODOs of `study.yaml`: summary (one sentence), description, findings, and the command of each run.
   Figures of the media folder can be used in the texts as `![caption](media:figures/comparison.png)`; the caption is
   shown under the figure, so say which runs it shows (and their line styles or colors). The figures of the whole
   study (`figures:`) can be given as `- path: figures/x.png` with a `caption:`. `thumbnail_caption` says what the
   thumbnail shows. Replace the thumbnail with `python tools/study.py thumbnail <slug> <image>`.
2. Upload the media folder `~/struphy-studies-media/<slug>/` to DataShare (web interface or desktop client).
3. On DataShare, share this folder with a link, remove the password, and copy the link. You can also set an expiry
   date on the link.
4. `python tools/study.py link <slug> <link>`: stores the link and checks that all media are there and served
   correctly.
5. Preview: `python tools/build_site.py --drafts && mkdocs serve`. Before the upload (steps 2-4), `--local-media`
   shows the media from the local media folder instead.
6. Commit and push (or open a pull request, if collaborators review the studies).

`draft: true` marks a study as unfinished: it is shown (marked as draft) on the site for the collaborators, but
`study.py publish` sets `draft: false` only in the public copy. `python tools/study.py check --linked` checks the
media of all studies with a link (CI does this on every pull request and weekly).

With [Claude Code](https://claude.com/claude-code), the skill `.claude/skills/create-study` guides through these steps.

## Share the site with collaborators

The collaborators need read access to the repository. Two ways:

**Download the built site.** Every push to `main` (and every pull request) builds the site with all studies, drafts
included, as the artifact `studies-site` of the run (kept 90 days). On GitHub: **Actions > Site >** the latest run
on `main` **> Artifacts > studies-site**, unzip it and open `index.html` in a browser. Or with the GitHub CLI:
```bash
run=$(gh run list -R <owner>/<name> -w Site -b main -s success -L 1 --json databaseId -q '.[0].databaseId')
gh run download $run -R <owner>/<name> -n studies-site -D studies-site
xdg-open studies-site/index.html     # macOS: open studies-site/index.html
```
The pages open from disk without a server; the figures and videos are loaded from DataShare (internet needed).
You can also send the zip to people without access to the repository.

**Clone and build locally.**
```bash
git clone https://github.com/<owner>/<name> && cd <name>
python -m venv .venv && .venv/bin/pip install -r requirements.txt && source .venv/bin/activate
python tools/build_site.py --drafts --offline    # site/index.html, opens from disk
# or, to browse with a local server:
python tools/build_site.py --drafts && mkdocs serve
```

## Publish a study

To make a study public, open it as a pull request in the public repository
[struphy-studies](https://github.com/struphy-hub/struphy-studies):
```bash
git clone https://github.com/struphy-hub/struphy-studies ../struphy-studies   # or your fork of it
python tools/study.py publish <slug> ../struphy-studies
```
`publish` checks that the media are complete on DataShare and that no TODOs are left, copies the committed files
of `studies/<slug>/` (untracked files are listed, not copied) into
the clone with `draft: false`, and prints the next steps (branch, local build, commit, pull request). Only that folder
becomes public: the history of your private repository stays private, and the media stay in the same DataShare
folder. Check `study.yaml` and `params/` for unpublished material, private paths and names before the pull request.
After the merge, the study is on the public site; you can keep or delete it in the private repository
(`publish --force` copies an updated version again).

## Update the tools

A repository created from the template does not follow the template. To take over newer tools:
```bash
git remote add template https://github.com/struphy-hub/struphy-studies-template   # once
git fetch template
git checkout template/main -- tools/ .github/ requirements.txt .claude/
git diff --staged    # review, then commit
```
This overwrites your own changes to these files (e.g. to the workflow) and does not delete files the template has
removed; `git diff --staged` shows both.

## Public collections

This template also works for a public collection with its own site: make the repository public, set
`site_url: https://<owner>.github.io/<name>/` in `mkdocs.yml`, choose **Settings > Pages > Source: GitHub Actions**,
and set the repository variable `DEPLOY_PAGES` to `true` (**Settings > Secrets and variables > Actions >
Variables**). Then pushes to `main` deploy the studies that are not drafts to GitHub Pages. (On GitHub Enterprise
Cloud, Pages of a private repository can be restricted to its members; remove the check
`!github.event.repository.private` in `.github/workflows/site.yml` to use that.)

## Reproducing a study

The study page gives the Struphy commit (`pip install "struphy @ git+https://github.com/struphy-hub/struphy@<commit>"`),
with its tags if it has any, and, for each run, the params file and the commands. Struphy records its version and
commit in the `run_metadata.json` of each run.
