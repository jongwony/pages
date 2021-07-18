date: 2019-08-21 01:00:00
title: AWS Lightsail Flask
tags: ['aws', 'lightsail', 'flask', 'nginx', 'waitress', 'install']
layout: post

```bash
sudo yum update -y
sudo yum upgrade -y

sudo yum install git -y
git clone https://github.com/jongwony/flask_blog.git
git config --global credential.helper store

curl -L https://raw.githubusercontent.com/pyenv/pyenv-installer/master/bin/pyenv-installer | bash
cat << EOF >> ~/.bashrc
export PATH="/home/ec2-user/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
EOF
source ~/.bashrc

# no acceptable C compiler found in $PATH
sudo yum install gcc -y
# ZipImportError: can't decompress data; zlib not available
sudo yum install zlib-devel -y
# No module named '_ctypes'
sudo yum install libffi-devel -y
# ERROR: The Python ssl extension was not compiled
sudo yum install openssl-devel -y
# WARNING: The Python bz2 extension was not compiled.
sudo yum install bzip2-devel
# WARNING: The Python readline extension was not compiled.
sudo yum install readline-devel
# WARNING: The Python sqlite3 extension was not compiled.
sudo yum install sqlite-devel

# latest 3.7.4
pyenv install --list
pyenv install 3.7.4
pyenv virtualenv 3.7.4 venv
pyenv shell venv

pip install --upgrade pip
pip install -r requirements.txt

# lxml option
sudo yum install libxml2-dev libxslt-dev
```

```
# https://flask.palletsprojects.com/en/1.1.x/tutorial/deploy/
export FLASK_APP=app
echo SECRET_KEY=`python -c 'import os; print(os.urandom(16))'` >> config.py
```

```
sudo yum install nginx -y
```

```nginx
location / {
    root                /home/ec2-user/flask_blog;

    proxy_pass          http://127.0.0.1:8080;
    proxy_redirect      off;

    proxy_set_header    Host                    $host;
    proxy_set_header    X-Real-IP               $remote_addr;
    proxy_set_header    X-Forwarded-For         $proxy_add_x_forwarded_for;
    proxy_set_header    X-Forwarded-Proto       $scheme;

}
# https://www.ionos.com/community/server-cloud-infrastructure/nginx/solve-an-nginx-403-forbidden-error/
location /static/ {
    root /home/ec2-user/flask_blog;
    autoindex on;
    autoindex_exact_size off;
}
```

```python
# wsgi.py
import sys

# add your project directory to the sys.path
project_home = u'/home/ec2-user/flask_blog'
if project_home not in sys.path:
    sys.path = [project_home] + sys.path

# import flask app but need to call it "application" for WSGI to work
from flask_blog.app import app as application  # noqa
```

```
waitress-serve --listen=127.0.0.1:8080 wsgi:application
```
