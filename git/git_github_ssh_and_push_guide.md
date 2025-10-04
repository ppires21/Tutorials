# 🚀 Push Local Git Repo to GitHub with SSH (Step by Step)

This guide explains how to correctly connect your local Git repo to GitHub using **SSH keys**, fix common errors, and push your code successfully.

---

## 1. Initialize Git (if not already done)
```bash
git init
```
👉 Creates a `.git` folder in your project directory.

---

## 2. Stage and Commit Your Files
```bash
git add .
git commit -m "first commit"
```
👉 `git add .` stages all files, and `git commit` saves them in Git history.

---

## 3. Generate an SSH Key (only once per machine)
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```
👉 Creates a new SSH key pair in `~/.ssh/id_ed25519`  
- `-t ed25519` = modern secure algorithm  
- `-C` = label with your email  

When prompted for file location, press **Enter** (default).  
When prompted for passphrase, you can leave it empty or set one.

---

## 4. Add the Key to ssh-agent
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```
👉 This ensures your system uses the key when connecting to GitHub.

---

## 5. Add the Public Key to GitHub
```bash
cat ~/.ssh/id_ed25519.pub
```
👉 Copy the entire output (starts with `ssh-ed25519 ...`).  

Then go to:  
**GitHub → Settings → SSH and GPG Keys → New SSH Key**  
Paste the copied key and save.

---

## 6. Test the SSH Connection
```bash
ssh -T git@github.com
```
✅ Expected result:
```
Hi <your-username>! You've successfully authenticated, but GitHub does not provide shell access.
```

---

## 7. Link Your Local Repo to GitHub
First, check if a remote already exists:
```bash
git remote -v
```
If the remote is missing or incorrect, set it with:
```bash
git remote set-url origin git@github.com:<your-username>/<your-repo>.git
```
👉 Replace `<your-username>` and `<your-repo>` with your GitHub details.

---

## 8. Push to GitHub
```bash
git push -u origin main
```
👉 Pushes your local branch `main` to GitHub.  

If GitHub created the repo with **master** instead of **main**, run:
```bash
git push -u origin main:master
```

---

## ✅ Done!
Now you can continue working with:
```bash
git add .
git commit -m "your message"
git push
```

Your code is now safely pushed to GitHub using SSH 🎉
