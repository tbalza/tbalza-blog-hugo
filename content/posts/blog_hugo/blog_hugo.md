---
title: "Deploy a Hugo-powered Blog on Github Pages using Github Actions"
date: "2024-02-28"
tags: ["git", "automation", "cost optimization"]
#categories: [""]
series: ["Hugo Blog"]
ShowToc: true
TocOpen: false
cover:
    image: "/posts/blog_hugo/cover.jpg"
    hiddenInSingle: true
summary: "In this article we will go through the steps of setting up a static website, that will not incur in storage costs or egress fees."
---

In this guide, we'll walk you through the process of creating a static website that won't burden you with storage costs or egress fees

We'll also help you set up a custom domain that will automatically reflect any changes made whenever a new commit is detected in the main branch of our version control system.

### The Problem with Egress Traffic

Whether you're operating an EC2 instance or directly serving files via S3, egress traffic can come with a price tag. For instance, AWS charges around $0.023 per GB. While this might not seem like much for personal pages, without the necessary precautions like DDoS protection or rate limiting, you could potentially face a hefty bill running into the thousands of dollars in the event of an attack. Fortunately, this won't be an issue with our setup.

![huge](/posts/blog_hugo/huge_bill.png)

#### The Solution: Free Tier Services

This is why it's a smart move to utilize services like GitHub Pages and CloudFlare for a public personal site. These platforms don't charge for outgoing traffic, even on their free tier service, making them an ideal solution.

#### Why Choose Hugo?

[Hugo](https://github.com/gohugoio/hugo) is a static site generator written in Go. It's not only incredibly fast but also simple to set up. The resulting pages load swiftly and are SEO optimized right from the start. This makes Hugo a fantastic alternative to platforms like WordPress, where publishing static content can be a bit of a hassle in terms of maintenance, costs, and performance.

![seo](/posts/blog_hugo/seo.png)

### Configure Your Personal Domain

- First, purchase your domain with services such as namecheap.com, then you can access the advanced settings and configure the records that will point to the GitHub Page that will reference to later

![namecheap](/posts/blog_hugo/namecheap.png)

- Set up the A Records and CNAME just like in the image above. Reference the [Github Pages Apex Configuration](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain) for further details. You can skip this step if you'd wish to deploy the site without a custom domain

### Set up Hugo

#### Install Hugo and Create a New Site

```bash
brew install hugo
hugo new site tbalza-blog-hugo -f yml
```

- Install Hugo with Homebrew

- Run hugo new site `<site_name>`. This will create a directory `<site_name>` containing the hugo templates. Pass in the `-f yml` argument so configuration files are stored in the yml format

#### Install Theme

```bash
git init
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git submodule update --init --recursive # needed when you reclone your repo (submodules may not get cloned automatically)
git submodule update --remote --merge
```
- Run git init on the root of the project to initialize a Git repository
- Install the [PaperMod theme](https://github.com/adityatelange/hugo-PaperMod) using Git Submodules

```yaml
theme: ["PaperMod"]
```
- Add the theme parameter in `hugo.yaml`

```yaml
baseURL: https://tbalza.net/
```
- Add the baseURL parameter in `hugo.yaml`

```bash
hugo server
```
- Run this command from within the project folder. This will create a blank page with no content

### Configure GitHub Actions to Publish to the GitHub Pages

#### Push the First Commit, Create gh-pages Branch

```bash
git commit -m "first commit"
git branch -M main # rename master to main
git remote add origin https://github.com/tbalza/tbalza-blog-hugo # replace with your repo
git push -u origin main
git branch gh-pages # used internally by github to execute the deployment action, will throw error if not created
```
- Create a repository on GitHub, add the remote address, and push your first commit, create gh-pages branch

#### Allow Read and Write Permissions

![read-write-permissions](/posts/blog_hugo/read-write-permissions.png)

- Under Settings > Actions > General > Workflow permissions, enable "Read and write permissions"

#### Set up Custom Domain in GitHub Pages

![github-custom-dns](/posts/blog_hugo/github-custom-dns.png)

- Configure your custom domain under Settings > Pages > Custom Domain

#### Create deploy.yml

```shell
mkdir -p .github/workflows
cd .github/workflows
touch deploy.yml
```
- In you project root folder create .github/workflows/deploy.yml

```yaml
name: Publish to GH Pages
on:
  push:
    branches:
      - main
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source
        uses: actions/checkout@v3
        with:
          submodules: true

      - name: Checkout destination
        uses: actions/checkout@v3
        if: github.ref == 'refs/heads/main'
        with:
          ref: gh-pages
          path: built-site

      - name: Setup Hugo
        run: |
          curl -L -o /tmp/hugo.tar.gz 'https://github.com/gohugoio/hugo/releases/download/v0.123.4/hugo_0.123.4_linux-amd64.tar.gz'
          tar -C ${RUNNER_TEMP} -zxvf /tmp/hugo.tar.gz hugo
          
      - name: Build
        run: ${RUNNER_TEMP}/hugo

      - name: Deploy
        if: github.ref == 'refs/heads/main'
        run: |
          cp -R public/* ${GITHUB_WORKSPACE}/built-site/
          cd ${GITHUB_WORKSPACE}/built-site
          git add .
          git config user.name 'tbalza'
          git config user.email 'tomas.balza@gmail.com'
          git commit -m 'Updated site'
          git push
```

- 1. **Checkout source** This action checks-out your repository under `$GITHUB_WORKSPACE`, so your workflow can access it.
`submodules:true` ensures that our submodule for the theme repository is fetched as well

- 2. **Checkout destination** The second step allows us to reference the `gh-pages` branch via the `$GITHUB_WORKSPACE/built-site` directory, where our static sites will be stored in

- 3. **Setup Hugo** This step downloads and extracts the Hugo static site generator. It uses `curl` to download a specific version of Hugo from its GitHub releases page, and `tar` to extract the downloaded file.

- 4. **Build** This step runs Hugo to build your static site. The built site's files are placed in the `public` directory.

- 5. **Deploy** This step deploys the built site to the `gh-pages` branch. It first copies the built site's files from the `public` directory to the `built-site` directory. It then changes the current directory to `built-site`, stages all changes for commit with `git add .`, sets the Git username and email, commits the changes with a message of 'Updated site', and finally pushes the commit to the `gh-pages` branch. 

The `gh-pages` branch is special because GitHub Pages serves the contents of this branch at your GitHub Pages URL. The `if: github.ref == 'refs/heads/main'` condition ensures that the site is only deployed when changes are pushed to the main branch.

![deploy](/posts/blog_hugo/deploy.png)

After the `deploy.yml` workflow is pushed, any future changes in the main branch, will trigger Hugo to generate an updated `/public/` folder and serve it in GitHub Pages.

