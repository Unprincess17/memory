# setup remote origin
```
git remote add src-name <remote-repository-url>
```

# Pull remote and overwrite local repository
```
git fetch src-name

git reset --hard src-name/branch-name
```

# Push local changes to overwrite remote repository
```
git push src-name branch-name --force-with-lease
```