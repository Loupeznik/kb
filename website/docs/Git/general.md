# General

## Change commit message

```powershell
git commit --amend
```

## Reset current commit

**Reset to HEAD~, keep changes**

```powershell
git reset --soft HEAD~
```

## Fix broken case-insensitive file/directory names

```powershell
git config core.ignorecase false
git rm -r --cached .
git add --all .
git status
git commit -a -m "Fixing file name casing"
git push origin master
```
