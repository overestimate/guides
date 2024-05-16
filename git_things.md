# Random Git Knowledge

## (De-)Anonymizing commit email
Get your GitHub email from [here](https://github.com/settings/emails), and make sure "Keep my email address private" is checked. The commit email should be after the text "We will instead use" near your primary email.
### Setting per-repo
Make sure to replace the placeholder email in the command. You can replace the email with a seperate email (such as your work email) if needed.
```
git config user.email "1337+REPLACEME@users.noreply.github.com"
```
### Setting the email globally
You should only do this if you primarily use GitHub.
```
git config --global user.email "1337+REPLACEME@users.noreply.github.com"
```
## Git Squash, Rebase, and Merge
### merge (`merge`)
Appends commit history of one branch to another via a commit
```
M=commit to merge
O=commit from main
X=commit from branch

main    - O - X - X M 
branch    \ - X - X |
```
### squash (`merge --squash`)
Squashes all the commits of a branch down into one and merges with another, basically merge without commit history
```
M=commit to merge
O=commit from main
X=commit from branch

main    - O - - - - M
branch    \ - X - X |
```
### rebase (`rebase`)
sets the base commit of the branch to the latest commit of the source
```
O=commit from main
X=commit from branch

main    - O - O - O -
branch    \ - X - X -
--rebase--
main    - O - O - O -   
branch            \ - X - X 
```
