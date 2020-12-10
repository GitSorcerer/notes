# notes
git config --global user.name "gaoh"
git config --global user.email "5146513+gaohwh@user.noreply.gitee.com"
创建 git 仓库:

mkdir notes
cd notes
git init
touch README.md
git add README.md
git commit -m "first commit"
git remote add mayun https://gitee.com/gaohwh/notes.git
git push -u mayun master
已有仓库?

cd existing_git_repo
git remote add mayun https://gitee.com/gaohwh/notes.git
git push -u mayun master

GitHub

…or create a new repository on the command line
echo "# notes" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M master
git remote add github https://github.com/GitSorcerer/notes.git
git push -u github master
                
…or push an existing repository from the command line
git remote add github https://github.com/GitSorcerer/notes.git
git branch -M master
git push -u github master

git config --global push.default simple