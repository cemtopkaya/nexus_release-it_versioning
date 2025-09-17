# release-it Deneme

## release-it Notları

Release-it’in kendi sağladığı değişkenler
Bunlar JSON config şablonlama sırasında otomatik dolduruluyor.
Resmî dokümana göre commitMessage, tagName, releaseName vb. yerlerde kullanılabilen değişkenler şunlar:

- `${name}` → `package.json` içindeki name
- `${version}` → çıkartılacak yeni sürüm
- `${latestVersion}` → önceki sürüm
- `${tagName}` → üretilen tag
- `${repo.host}`, `${repo.owner}`, `${repo.project}` vs. → repo bilgileri
- `${changelog}` → git changelog çıktısı


### 1. Nexus admin kullanıcısına yeni şifre atayalım

```sh
otomatikSifre=$(docker exec nexus cat /nexus-data/admin.password)

curl -vvv -X 'PUT' -u admin:$otomatikSifre \
  'http://nexus:8081/service/rest/v1/security/users/admin/change-password' \
  -H 'accept: application/json' \
  -H 'Content-Type: text/plain' \
  -d '111'
```


### 2. Yeni Kullanıcı Tanımlayalım

```sh
curl -X POST http://nexus:8081/service/rest/v1/security/users \
    -vvv \
    -u admin:111 \
    -H "Content-Type: application/json" \
    -d '{
  "userId": "cem",
  "firstName": "Cem",
  "lastName": "Topkaya",
  "emailAddress": "cem@example.com",
  "password": "111",
  "status": "active",
  "roles": [
    "nx-admin",
    "nx-anonymous"
  ]
}'
```

### 3. Yeni NPM Registry Tanımlayalım

```sh
curl -X 'POST' \
  -u admin:111 \
  'http://nexus:8081/service/rest/v1/repositories/npm/hosted' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "name": "npm-paket-deposu",
  "online": true,
  "storage": {
    "blobStoreName": "default",
    "strictContentTypeValidation": true,
    "writePolicy": "allow_once"
  },
  "cleanup": {
    "policyNames": [
    ]
  },
  "component": {
    "proprietaryComponents": true
  }
}' \
 -vvv
```

### 4. Yeni NPM Kütüğümüzün Bilgilerini Çekelim

```sh
curl -u admin:111 \
  'http://nexus:8081/service/rest/v1/repositories/npm/hosted/npm-paket-deposu' \
  -H 'accept: application/json' \
  -vvv
```

### 5. Realm İçine "npm Bearer Token Realm" Eklenir

```sh
curl 'http://nexus:8081/service/rest/v1/security/realms/active' \
  -X 'PUT' \
  -u admin:111 \
  -H 'Content-Type: application/json' \
  --data-raw '["NpmToken","NexusAuthenticatingRealm"]' \
  -vvv
```

### 6. npmrc Dosyasına Yazalım

```sh
: > .npmrc

# sürekli npm kütüğünün adresini komutlara geçmemek için .npmrc dosyasına yazalım:
echo "registry=https://registry.npmjs.org/" >> .npmrc
echo "@ornekKapsam:registry=http://nexus:8081/repository/npm-paket-deposu/" >> .npmrc

# "npm add-user" Komutuyla kullanıcı adı, şifresi, eposta adresini girmemek için .npmrc'ye yazalım:
BASE64_ENCODED_USERNAME_PASSWORD=$(echo -n 'cem:111' | base64)
echo "//nexus:8081/repository/npm-paket-deposu/:username=cem" >> .npmrc
echo "//nexus:8081/repository/npm-paket-deposu/:_auth_=$BASE64_ENCODED_USERNAME_PASSWORD" >> .npmrc
echo "//nexus:8081/repository/npm-paket-deposu/:email=cem@example.com" >> .npmrc
echo "//nexus:8081/repository/npm-paket-deposu/:always-auth=true" >> .npmrc
```

NPM Kullanıcı girişi (`--scope`):
```sh
npm login --scope @ornekKapsam
```

NPM Kullanıcı girişi (`--registry`):
```sh
npm login --registry http://nexus:8081/repository/npm-paket-deposu/
```

# Git İşlemleri

### .gitmessage İle commit mesajı oluşturma

```sh
cat > .gitmessage <<EOF
# ===========================================
# JIRA commit mesajı standartına göre:
# 
# Satır 1: Jira bilet numarası: kısaca ne yaptığımızı açıklayan bir kısaca açıklama.
# Satır 2: Boş bırakılacak
# Satır 3: Tafsilat burada olacak. Madde madde bu değişimi
# - neden yaptık, 
# - neyi etkiledi,
# - gerekiyorsa ek bilgiler
# ===========================================

# Line 1: JIRA ID + short title (max 50 characters)
# Example: PROJ-123: Fixed login error

<ISSUE-ID>: <Short and concise title>

# -------------------------------------------
# Line 2: Leave empty
# -------------------------------------------

# Line 3 and onwards: More detailed description (optional)
# - Why was this change made?
# - Which components are affected?
# - Add any additional notes if necessary

# Tip: Lines starting with '#' will not be included in the commit
EOF
```