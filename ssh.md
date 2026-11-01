1.generate key
ssh-keygen -t ed25519 -C "email@example.com"
2.copy ssh key
cat ~/.ssh/id_ed25519.pub
copy all
3.add it to github
Settings->SSH and GPG keys->New SSH key
4.try link
ssh -T git@github.com



