# Sloth Blog

This is a blog project designed for lazy people, relying on Github Page and GitHub Action to achieve automatic deployment.

这是一个为懒人设计的博客项目, 依托于Github Page和GitHub Action实现自动化部署. 

If the user has no customization requirements, he only needs to complete the basic configuration and install the Git environment locally.

使用者如果没有定制化需求, 仅需完成基本配置, 在本地安装好Git环境即可使用.

**ExampleSite** can be found here: []()

### How it works

First, let me introduce the project organization, there are five main directory in this project:

- `.github`: You need to configure here to ensure that your blog files can be automatically processed and deployed.

- `content`: This is where your blog files be. 

- `config`: There are two config file which one for hugo-blog-builder and the other for blog-processor.

- `hugo-blog-builder`: This is the submodule [hugo-blog-builder](https://github.com/RoaraeonLiou/hugo-blog-builder) .

- `blog-processor`: This is the submodule [blog-processor](https://github.com/RoaraeonLiou/blog-processor).
  
  
Next, we will introduce how the project works along the configuration of the workflow.
```yaml
# This is your action name, you can name it whatever you want. 
name: From Shadow To The Land

# This define when and on which branch this action will be execute. 
on:
  push:
    branches: [ main ]

# Now let's look how the jobs define
jobs: 

  # First setup the os 
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      # 1. Checkout this branch, just as this step's name says. 
      - name: Checkout this branch 
        uses: actions/checkout@v4
        with:
          persist-credentials: false
          fetch-depth: 0

      # 2. Update submodules, this is IMPORTANT. 
      #    Because simply pulling a branch will not pull the content in the submodule, you need to pull it manually.
      - name: Update submodules
        run: git submodule update --init --recursive

      # 3. Setup hugo, because this project is based on hugo
      - name: Setup hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
      
      # 4. Pull the static branch of your io repository.
      #    Because we store static files and sqlite database files in the static branch. 
      #    Make sure your io repository has a static branch.
      #    Here you need to change the repository address to your own.
      - name: Pull io-repo's static branch
        uses: actions/checkout@v4
        with:
          repository: github_name/github_name.github.io
          ref: static
          path: static
          token: ${{ secrets.BLOG_ACCESS_TOKEN }}

      # 5. Now it's the showtime of blog-processor.
      #    First, grant the packaged file execution permission.
      #    And overwrite the default configuration file of blog-processor with yours. 
      #    Then execute blog-processor, and it will process your blog files.
      #    This submodule will replace the image source, add header content, etc. to the markdown file according to your configuration.
      - name: Processing blog files 
        run: |
          chmod +x ./blog-processor/processor
          cp -rf ./config/config.yaml ./blog-processor/
          cd blog-processor
          ./processor
          cd ..

      # 6. Then the hugo-blog-builder comes to build your static website.
      #    First overwrite the default configuration file.
      #    Then copy the theme package file and layouts file to submodule.
      #    Finally, build your own static website through hugo.
      - name: Build static files
        run: |
          cp -rf ./config/hugo.yaml ./hugo-blog-builder/
          cp -rf ./themes/* ./hugo-blog-builder/themes/
          cp -rf ./layouts/* ./hugo-blog-builder/layouts/
          cd hugo-blog-builder
          bash ./run.sh
      
      # 7. Check if there are any changes in the static branch
      - name: Check for changes of static branch
        working-directory: ./static
        id: check_static_changes
        run: |
          git add -A
          if git diff --cached --quiet; then
            echo "No changes to commit"
            echo "::set-output name=changes::false"
          else
            echo "Changes found"
            echo "::set-output name=changes::true"
          fi

      # 8. If there are changes in the static branch, push it.
      #    But please note that you need to configure GitHub token here. 
      #    And set the corresponding repository link to your own.
      - name: Commit and push changes for static branch
        working-directory: ./static
        if: steps.check_static_changes.outputs.changes == 'true'
        run: |
          git -c user.name='github actions by ${{ github.actor }}' -c user.email='NO' commit -m 'update' 
          git push "https://${{ secrets.GITHUB_TOKEN }}@github.com/github_name/github_name.github.io.git" HEAD:static -f -q

      # 9. Force push to the main branch. 
      #    Here you also need to change the repository link to your own.
      - name: Commit and Push to main branch
        working-directory: ./hugo-blog-builder/public
        run: |
          pwd
          git init
          git checkout -b master
          git add -A
          git -c user.name='github actions by ${{ github.actor }}' -c user.email='NO' commit -m 'update' 
          git push "https://${{ secrets.GITHUB_TOKEN }}@github.com/github_name/github_name.github.io.git" HEAD:main -f -q
```

### Feature

- 
