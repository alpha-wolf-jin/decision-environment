# decision-environment

**Prepare GIT**
```
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/alpha-wolf-jin/decision-environment.git

git config --global credential.helper 'cache --timeout 72000'
git push -u origin main

git add . ; git commit -a -m "update README" ; git push -u origin main
````

# Base image

**Start container from base image**
```
# podman run -ti --name de-01 --hostname de-01 --network host registry.redhat.io/ansible-automation-platform-24/de-supported-rhel9:1.0.0-68  /bin/bash
```

**Verify Java version**
```
bash-5.1# java -version
openjdk version "17.0.7" 2023-04-18 LTS
OpenJDK Runtime Environment (Red_Hat-17.0.7.0.7-2) (build 17.0.7+7-LTS)
OpenJDK 64-Bit Server VM (Red_Hat-17.0.7.0.7-2) (build 17.0.7+7-LTS, mixed mode, sharing)
```

**Verify python libraries**
```
bash-5.1# pip3 freeze
aiobotocore==2.5.0
aiodns==3.0.0
aiohttp==3.8.1
aioitertools==0.11.0
aiokafka==0.8.0
aiosignal==1.2.0
ansible-core==2.15.0
ansible-rulebook==1.0.0
ansible-runner==2.3.2
async-timeout==4.0.2
attrs==21.4.0
azure-core==1.26.4
azure-servicebus==7.9.0
botocore==1.29.76
Brotli==1.0.9
cchardet==2.1.7
certifi==2022.12.7
cffi==1.15.0
charset-normalizer==2.0.12
cryptography==38.0.4
Deprecated==1.2.13
docutils==0.16
dpath==2.1.4
drools-jpy==0.3.4
frozenlist==1.3.0
idna==3.3
importlib-resources==5.0.7
isodate==0.6.1
janus==1.0.0
Jinja2==3.1.2
jmespath==0.9.4
jpy==0.13.0
jsonschema==4.16.0
kafka-python==2.0.2
lockfile==0.12.2
MarkupSafe==2.1.0
multidict==6.0.2
oauthlib==3.1.1
packaging==21.3
pexpect==4.8.0
ptyprocess==0.6.0
pycares==4.1.2
pycparser==2.21
pyparsing==3.0.9
pyrsistent==0.17.3
python-daemon==2.3.0
python-dateutil==2.8.1
PyYAML==5.4.1
redis==4.3.4
requests==2.28.2
requests-oauthlib==1.3.0
resolvelib==0.5.4
six==1.16.0
typing_extensions==4.5.0
uamqp==1.6.4
urllib3==1.26.8
watchdog==3.0.0
websockets==10.4
wrapt==1.14.1
yarl==1.8.2
bash-5.1# 
```

**Verify Python Version**
```
bash-5.1# python3 --version
Python 3.9.16
```

**Verify required rpm**
```
bash-5.1# rpm -q gcc python3-devel python3-pip python3-systemd systemd-devel 
package gcc is not installed
package python3-devel is not installed
python3-pip-21.2.3-6.el9.noarch
package python3-systemd is not installed
package systemd-devel is not installed
```

**Reuse existing container :  Start the stopped container**
```
# podman ps -a
CONTAINER ID  IMAGE                                                                          COMMAND     CREATED         STATUS                     PORTS       NAMES
baa5fec4911f  localhost/rulebas-01:latest                                                    /bin/bash   4 days ago      Exited (143) 4 days ago                rulebase-02
536da32c923a  registry.redhat.io/ansible-automation-platform-24/de-supported-rhel9:1.0.0-68  /bin/bash   11 minutes ago  Exited (0) 19 seconds ago              de-01

[root@aap-eda decision-environment]# podman start 536da32c923a
536da32c923a

[root@aap-eda decision-environment]# podman ps
CONTAINER ID  IMAGE                                                                          COMMAND     CREATED         STATUS        PORTS       NAMES
536da32c923a  registry.redhat.io/ansible-automation-platform-24/de-supported-rhel9:1.0.0-68  /bin/bash   13 minutes ago  Up 6 seconds              de-01
```
**Reuse existing container :  access running container as root**
```
# podman exec -it 536da32c923a /bin/bash 
bash-5.1# 

bash-5.1# exit
exit

# podman exec -it --user 0 536da32c923a /bin/bash 
bash-5.1# 
```

# Ansible builder definition file for Kafka

```
# cat de-kafka-02.yaml 
version: 3

images:
  base_image:
    name: 'registry.redhat.io/ansible-automation-platform-24/de-supported-rhel9:1.0.0-68'

dependencies:
  galaxy:
    collections:
      - ansible.eda
  python:
    - aiokafka
    - wheel
  system:
    - python3-systemd [platform:rpm]
    - systemd-devel [platform:rpm]
    - gcc [platform:rpm]
    - python3-devel [platform:rpm]
    - python3-pip [platform:rpm]
  python_interpreter:
    package_system: "python39"

options:
  package_manager_path: /usr/bin/microdnf

```

In order to make ansible-builder work well, we need to add 'python3-systemd', 'systemd-devel', 'gcc', 'python3-devel', and 'python3-pip'. 

What the Kafka needs is python library 'aiokafka'.


# Ansible builder de image for Kafka

```
# ansible-builder build -f de-kafka-01.yaml -t de-kafka-01:latest
Running command:
  podman build -f context/Containerfile -t de-kafka-01:latest context
Complete! The build context can be found at: /root/ansi/decision-environment/context

```

# Things to note

The `additional_build_steps` should work, but not at this moment.

```
...
additional_build_steps: 
  prepend: |
    RUN whoami
    RUN cat /etc/os-release
  append:
    - RUN echo This is a post-install command!
    - RUN ls -la /etc
...
```

Some rpm package is not available, for exmaple: tmux 

## Work Around

When `ansible-builder` cannot meet the requirement, we can look into podman build.

Directly modify `context` directory and ./context/Containerfile file

**Copy rpm Package**
```
# ll context/_rpm
total 480
-rw-r--r--. 1 root root 487866 Jul 14 13:11 tmux-3.2a-4.el9.x86_64.rpm
```

**Insert actions into ./context/Containerfile**
```
COPY _rpm/tmux-3.2a-4.el9.x86_64.rpm tmux-3.2a-4.el9.x86_64.rpm
RUN rpm -ivh tmux-3.2a-4.el9.x86_64.rpm
RUN rm -f tmux-3.2a-4.el9.x86_64.rpm
```

Inset to the end portion
```
COPY --from=builder /output/ /output/
RUN /output/scripts/install-from-bindep && rm -rf /output/wheels
RUN chmod ug+rw /etc/passwd
RUN mkdir -p /runner && chgrp 0 /runner && chmod -R ug+rwx /runner
WORKDIR /runner
RUN $PYCMD -m pip install --no-cache-dir 'dumb-init==1.2.5'
RUN rm -rf /output
COPY _rpm/tmux-3.2a-4.el9.x86_64.rpm tmux-3.2a-4.el9.x86_64.rpm
RUN rpm -ivh tmux-3.2a-4.el9.x86_64.rpm
RUN rm -f tmux-3.2a-4.el9.x86_64.rpm
LABEL ansible-execution-environment=true
USER 1000
ENTRYPOINT ["/opt/builder/bin/entrypoint", "dumb-init"]
CMD ["bash"]
```

**Use Podman to builder image**
```
# podman build -f context/Containerfile -t de-kafka-02:latest context
...
[4/4] STEP 18/24: COPY _rpm/tmux-3.2a-4.el9.x86_64.rpm tmux-3.2a-4.el9.x86_64.rpm
--> 5bf5cc3d1b6
[4/4] STEP 19/24: RUN rpm -ivh tmux-3.2a-4.el9.x86_64.rpm
Verifying...                          ########################################
Preparing...                          ########################################
Updating / installing...
tmux-3.2a-4.el9                       ########################################
--> 69e01adf472
[4/4] STEP 20/24: RUN rm -f tmux-3.2a-4.el9.x86_64.rpm
--> 2483013a76e
[4/4] STEP 21/24: LABEL ansible-execution-environment=true
--> e7e4ef40d39
[4/4] STEP 22/24: USER 1000
--> dcb59c51eec
[4/4] STEP 23/24: ENTRYPOINT ["/opt/builder/bin/entrypoint", "dumb-init"]
--> de9e87b0206
[4/4] STEP 24/24: CMD ["bash"]
[4/4] COMMIT de-kafka-02:latest
--> 2c5b6bd4105
Successfully tagged localhost/de-kafka-02:latest
2c5b6bd4105db4c1c982360c53e56cae02426177795a367e8f5bc0e93e123bab

```
