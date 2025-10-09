```
apt-get update && apt-get install samba-dc wget dos2unix -y
wget https://raw.githubusercontent.com/sudo-project/sudo/main/docs/schema.ActiveDirectory
dos2unix schema.ActiveDirectory
sed -i 's/DC=X/DC=test,DC=alt/g' schema.ActiveDirectory
```
