### rebase使用

#### 使用rebase合并commit

假设当前分支有一下commit
	* alter log 
	* alter log format
	* add log
	* add feature 1234

最新的三个commit重复，不利于review，并且时git tree比较混乱。此时可以使用rebase合并分支。假设“add feature 1234”的commit SHA为abc123，则使用
```
git rebase -i abc123
```
可以合并commit，使用命令后会进入到一个vi的编辑界面，将"alter log"和"alter log format"两个commit的前缀修改为squash，然后保存退出，会进入到另一个vi编辑界面，可以编辑合并后的commit信息。


#### 使用rebase合并分支

假设有以下场景：

我们从master分支上切出了feature分支进行开发，此时commit tree如下:

```
master: C1->C2
feature: C1->C2
```

然后在feature的开发过程中，有同事在master分支上修改了bug，提交了hotfix，commit tree变为

```
master: C1->C2->C3->C4
feature: C1->C2->C5->C6
```
这时我们想要在最新的hotfix的master分支上再开发，此时我们可以选择在feature分支git merge master，但是这样feature分支上就有merge的commit log，当feature合并到master时，就会有冗余的commit log污染commit tree。

这时候rebase就派上用场了，我们可以在feature分支上执行git rebase master，git会将feature分支上我们的提交C5,C6保存起来，然后将feature分支“变基”到master的最新版本，之后将C5，C6再依次应用的feature分支。

如果中途有conflict，则需要手动修复conflict并add冲突文件然后执行git rebase --continue，git会继续合并进程，如果不想合并了，可以使用git rebase --abort来使分支回到rebase之前的状态

rebase之后直接push，git可能会拒绝，因为远程分支与本地不一致，此时push -f即可

