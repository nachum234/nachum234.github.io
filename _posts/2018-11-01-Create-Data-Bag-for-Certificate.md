---
layout: post
title: "Create Data Bag for Certificate"
date: 2018-11-01
---

It is not secure and you should use something like vault but if you still want to migrate certificate file to data bag you can use the following:

1. Replace new lines with /n  
```
cat fullchain.pem | tr '\n' '#' | sed 's/#/\\n/g'
```

2. Do the same thing for your key file  
```
cat key.pem | tr '\n' '#' | sed 's/#/\\n/g'
```

3. Copy the data to the data bag
