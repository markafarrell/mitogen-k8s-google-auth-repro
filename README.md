### Reproduction Instructions

0. Install virtualenv and ssh pass

   ```
   pip3 install virtualenv
   apt install sshpass
   ```

1. Create Virtual Environment

   ```virtualenv -p python3 .venv```

2. Enter Virtual Environment

   ```source .venv/bin/activate```

3. Install python requirments

   ```pip3 install -r requirements.txt```

4. Install ansible requirements

   ```ansible-galaxy collection install -p collections -r requirements.yml```

5. Start test infrastructure

    ```
    docker network create k8s;
    docker volume create k8s-config;

    docker run -dt --name management-server \
        -e KUBECONFIG=/data/kubeconfig.yaml \
        -v k8s-config:/config:ro \
        -v $PWD/management:/data \
        -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
        --network=k8s \
        --privileged \
        --rm \
        geerlingguy/docker-debian12-ansible:latest;
    docker run -dt --name k3s \
        -e K3S_KUBECONFIG_OUTPUT=/config/kubeconfig.yaml \
        -e K3S_KUBECONFIG_MODE="666" \
        -e K3S_TOKEN="123456" \
        -v k8s-config:/config:rw \
        --network=k8s \
        --privileged \
        --rm \
        rancher/k3s:v1.27.4-k3s1 server --tls-san=k3s;
    ```

6. Setup Management Server

    ```
    docker exec management-server apt update;
    docker exec management-server apt install -y openssh-server python3 curl;
    docker exec management-server systemctl enable sshd;
    docker exec management-server systemctl start sshd;
    docker exec management-server pip3 install --upgrade -r /data/management-requirements-broken.yml;

    # Fix kubeconfig
    docker exec management-server cp /config/kubeconfig.yaml /data/
    docker exec management-server sed -i 's/127[.]0[.]0[.]1/k3s/' /data/kubeconfig.yaml

    # Install kubectl
    docker exec management-server curl -LO "https://dl.k8s.io/release/v1.27.4/bin/linux/amd64/kubectl"
    docker exec management-server install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    docker exec -e no_proxy=k3s -e NO_PROXY=k3s management-server kubectl get nodes

    # Add test user
    docker exec -i management-server useradd --create-home test;

    # Set test user password
    printf 'test123456\ntest123456' | docker exec -i management-server passwd test;

    # Copy kubeconfig into test user home
    docker exec -i management-server mkdir -p /home/test/.kube;
    docker exec -i management-server cp /data/kubeconfig.yaml /home/test/.kube/config;
    docker exec -i management-server chown test:test /home/test/.kube/config;
    ```

7. Run playbook with normal strategy

   ```ansible-playbook playbook.yml \
    -i ",$(docker inspect -f \
        '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
        management-server \
    )" \
    --extra-vars=strategy=linear \
    --extra-vars=ansible_user=test \
    --extra-vars=ansible_password=test123456
    ```

8. Run playbook with mitogen strategy

   ```ansible-playbook playbook.yml \
    -i ",$(docker inspect -f \
        '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
        management-server \
    )" \
    --extra-vars=strategy=mitogen_linear \
    --extra-vars=ansible_user=test \
    --extra-vars=ansible_password=test123456
    ```

    This fails

    ```
    [WARNING]: Found variable using reserved name: strategy

    PLAY [Create a namespace] *********************************************************************************************************************************************************************************************

    TASK [Creating namespace] *********************************************************************************************************************************************************************************************
    fatal: [172.24.0.2]: FAILED! => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python3"}, "changed": false, "msg": "Failed to import the required Python library (kubernetes) on 3c295df4a8a8's Python /usr/bin/python3. Please read the module documentation and install it in the appropriate location. If the required library is installed, but Ansible is using the wrong Python interpreter, please consult the documentation on ansible_python_interpreter"}

    PLAY RECAP ************************************************************************************************************************************************************************************************************
    172.24.0.2                 : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
    ```

9. Downgrade google-auth
    ```
    docker exec management-server pip3 install --upgrade -r /data/management-requirements-working.yml;
    ```

10. Re-run with mitogen enabled

    ```ansible-playbook playbook.yml \
    -i ",$(docker inspect -f \
        '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
        management-server \
    )" \
    --extra-vars=strategy=mitogen_linear \
    --extra-vars=ansible_user=test \
    --extra-vars=ansible_password=test123456
    ```

    This works

    ```
    [WARNING]: Found variable using reserved name: strategy

    PLAY [Create a namespace] *****************************************************************************************************************************************************************************

    TASK [Creating namespace] *****************************************************************************************************************************************************************************
    ok: [172.24.0.2]

    TASK [Get namespace] **********************************************************************************************************************************************************************************
    ok: [172.24.0.2]

    TASK [debug] ******************************************************************************************************************************************************************************************
    ok: [172.24.0.2] => {
        "test": {
            "ansible_facts": {
                "discovered_interpreter_python": "/usr/bin/python3"
            },
            "api_found": true,
            "changed": false,
            "failed": false,
            "resources": [
                {
                    "apiVersion": "v1",
                    "kind": "Namespace",
                    "metadata": {
                        "creationTimestamp": "2023-08-03T04:50:00Z",
                        "labels": {
                            "kubernetes.io/metadata.name": "test"
                        },
                        "managedFields": [
                            {
                                "apiVersion": "v1",
                                "fieldsType": "FieldsV1",
                                "fieldsV1": {
                                    "f:metadata": {
                                        "f:labels": {
                                            ".": {},
                                            "f:kubernetes.io/metadata.name": {}
                                        }
                                    }
                                },
                                "manager": "OpenAPI-Generator",
                                "operation": "Update",
                                "time": "2023-08-03T04:50:00Z"
                            }
                        ],
                        "name": "test",
                        "resourceVersion": "793",
                        "uid": "d317a1ea-3f22-41ff-8d72-ff8fc8071c1c"
                    },
                    "spec": {
                        "finalizers": [
                            "kubernetes"
                        ]
                    },
                    "status": {
                        "phase": "Active"
                    }
                }
            ]
        }
    }

    PLAY RECAP ********************************************************************************************************************************************************************************************
    172.24.0.2                 : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    ```
