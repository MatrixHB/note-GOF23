#### 在IDEA上传代码到github

1. IDEA --> settings --> github中添加自己的账户，git中test一下
2. VCS --> Import into Version Control --> Create Git Repository 选择本地仓库位置，比如项目根目录下，创建完成后会生成一个.git文件
3. 提交代码时，project上右键，git --> add，其次commit，其次Repository --> push
4. 如果出现push rejected错误，可能因为远程origin分支和本地分支的代码有不相干的提交历史，则
    在项目文件夹中打开git bash，执行语句`git pull origin master --allow-unrelated-histories`
       此时会将远程origin分支上的代码拉下来进行merge，也就是修改的合并，然后再push就能成功
5. get pull = git fetch + git merge



#### 在文件夹上传代码到github

1、打开git bash -->  输入命令`git init` （创建一个本地仓库，会出现.git文件，并有一个master分支）

2、写好文件之后，`git add --all`  -->   `git commit -m 'change something'`（在本地提交修改）

3、需要建立和github的SSH连接，输入命令`ssh-keygen -t rsa`（在/c/users/xxx/.ssh/文件夹中出现一个公钥文件和一个私钥文件），将公钥加入github --> settings --> SSH and GPG keys

4、`git remote add origin git@github.com:MatrixHB/test.git` 添加远程仓库origin

5、`git push origin master`，推送代码到github仓库

