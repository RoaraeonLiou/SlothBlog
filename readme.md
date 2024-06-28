<!-- # Sloth Blog -->
<h1 align=center> 🦥 Sloth Blog 🦥 </h1>

<h4 align=center> 🍳Easy | 😃Simple | ☕️Low Code </h4>

<h5 align=center> Yeah ~, Just 🍴 Fork Me Up...</h5>

This is a blog project designed for lazy people, relying on Github Page and GitHub Action to achieve automatic deployment.
If the user has no customization requirements, he only needs to complete the basic configuration and install the Git environment locally. 
<!-- 这是一个为懒人设计的博客项目, 依托于Github Page和GitHub Action实现自动化部署.  -->
<!-- 使用者如果没有定制化需求, 仅需完成基本配置, 在本地安装好Git环境即可使用. -->

**🧑‍💻 Example Site** can be found here: [My Blog](https://roaraeonliou.github.io/)

**📖 Step-by-step Tutorial** can be found here: [Tutorial](https://roaraeonliou.github.io/tutorial/how_to_build_this_blog_website/)

### Feature

- 👐 Completely open source, including submodules.
- 🎉 Supports directory-level header file definition. Header files support yaml, toml, and json. The default is toml. If you need to change it, please change it in config/config.yaml.
- 🤖 Automatically upload local images to github after MD5 encryption and confusion, and replace the image source in markdown.
- 🦾 Added 5 new header fields to make blogging easier
  - 📝 `status`: String type. When this field is different from the last time, blog-processor will update this blog. If the value is set to "update", blog-processor will strongly update this blog every time.
  - 🏷️ `include_tags` and `exclude_tags`: List of strings used to add and exclude tags defined at the directory level.
  - 🗂️ `include_categories` and `exclude_categories`: List of strings used to add and exclude categories defined at the directory level.
- 🏠 Supports custom themes, just copy the theme file and layout file to the `themes` folder and `layouts` folder.


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
      - name: Pull io-repo's static branch
        uses: actions/checkout@v4
        with:
          repository: ${{ secrets.USERNAME }}/${{ secrets.USERNAME }}.github.io
          ref: static
          path: static
          token: ${{ secrets.BLOG_TOKEN }}

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
      #    Then copy the theme package files, layouts files and assets files to submodule.
      #    Finally, build your own static website through hugo.
      - name: Build static files
        run: |
          if [ -e "./config/hugo.yaml" ]; then
            cp -rf ./config/hugo.yaml ./hugo-blog-builder/
          fi
          if [ -e "./themes" ]; then
            cp -rf ./themes/* ./hugo-blog-builder/themes/
          fi
          if [ -e "./layouts" ]; then
            cp -rf ./layouts/* ./hugo-blog-builder/layouts/
          fi
          if [ -e "./assets" ]; then
            cp -rf ./assets ./hugo-blog-builder/
          fi
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
      - name: Commit and push changes for static branch
        working-directory: ./static
        if: steps.check_static_changes.outputs.changes == 'true'
        run: |
          git -c user.name='github actions by ${{ github.actor }}' -c user.email='NO' commit -m 'update' 
          git push "https://${{ secrets.BLOG_TOKEN }}@github.com/${{ secrets.USERNAME }}/${{ secrets.USERNAME }}.github.io.git" HEAD:static -f -q

      # 9. Force push to the main branch. 
      - name: Commit and Push to main branch
        working-directory: ./hugo-blog-builder/public
        run: |
          pwd
          git init
          git checkout -b master
          git add -A
          git -c user.name='github actions by ${{ github.actor }}' -c user.email='NO' commit -m 'update' 
          git push "https://${{ secrets.BLOG_TOKEN }}@github.com/${{ secrets.USERNAME }}/${{ secrets.USERNAME }}.github.io.git" HEAD:main -f -q
```

After that, configure hugo's configuration files `hugo.yaml` and blog-prosessor's configuration files `config.yaml` in the `config` directory. The main thing is to replace the GitHub repository address with your own. Other configurations can be used by default.

Finally, just write the blog file in the content directory and submit it through the git command to automatically deploy it to github pages.

### Support
- Star 🌟 this repository.
- Help spread SlothBlog project by sharing it on social media and recommending it to your friends. 

### Special Thanks
- [hugo](https://github.com/gohugoio/hugo)
- [PaperMod](https://github.com/adityatelange/hugo-PaperMod/tree/master)
- [Lovely you]()

