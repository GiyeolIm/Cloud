apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  creationTimestamp: '2021-05-07T08:33:38Z'
  generation: 14
  name: copy-war-jh
  resourceVersion: '54591763'
  selfLink: /apis/tekton.dev/v1beta1/clustertasks/copy-war-jh
  uid: 0f320fd3-7b27-4615-a0a5-af1904930c13
spec:
  description: copy-war-task
  params:
    - description: Image to be saved
      name: IMAGE_URL
      type: string
    - default: .
      description: The location of the path to run s2i from.
      name: PATH_CONTEXT
      type: string
    - default: ''
      description: Docker registry secret (kubernetes.io/dockerconfigjson type)
      name: REGISTRY_SECRET_NAME
      type: string
    - default: 'false'
      description: >-
        Verify the TLS on the registry endpoint (for push/pull to a non-TLS
        registry)
      name: TLSVERIFY
      type: string
    - default: '0'
      description: Log level when running the S2I binary
      name: LOGLEVEL
      type: string
    - default: ''
      description: >-
        URL (including protocol, ip, port, and path) of private package server
        (e.g., devpi, pypi, verdaccio, ...)
      name: PACKAGE_SERVER_URL
      type: string
  results:
    - description: Tag-updated image url
      name: image-url
    - description: Tag-updated image url
      name: registry-cred
  steps:
    - args:
        - update-image-url
      env:
        - name: SOURCE_PATH
          value: $(workspaces.git-source.path)
        - name: IMAGE_URL
          value: $(params.IMAGE_URL)
        - name: TARGET_FILE
          value: $(results.image-url.path)
      image: '192.168.178.91:5000/cicd-util:1.2.1'
      name: update-image-url
      resources: {}
    - args:
        - parse-registry-cred
      env:
        - name: SECRET_NAME
          value: $(params.REGISTRY_SECRET_NAME)
        - name: IMAGE_URL_FILE
          value: $(results.image-url.path)
        - name: TARGET_FILE
          value: $(results.registry-cred.path)
      image: '192.168.178.91:5000/cicd-util:1.2.1'
      name: parse-registry-cred
      resources: {}
    - image: '192.168.178.91:5000/busybox:v0.12.1'
      name: generate
      resources: {}
      script: >
        #!/bin/sh

        set -ex

        touch /gen-source/Dockerfile

        mv /workspace/git-source/ROOT.war /gen-source/ROOT.war

        echo FROM 192.168.178.91:5000/tomcat-test:8b48f06 >>
        /gen-source/Dockerfile

        echo RUN rm -rf /tomcat/webapps/ROOT* >> /gen-source/Dockerfile

        echo RUN mkdir -p /tomcat/webapps/ROOT/ >> /gen-source/Dockerfile

        echo COPY ROOT.war /tomcat/webapps/ROOT/ >> /gen-source/Dockerfile

        echo WORKDIR /tomcat/webapps/ROOT/ >> /gen-source/Dockerfile

        echo RUN jar -xvf ROOT.war >> /gen-source/Dockerfile

        echo RUN rm -rf ROOT.war >> /gen-source/Dockerfile
      volumeMounts:
        - mountPath: /gen-source
          name: gen-source
      workingDir: $(workspaces.git-source.path)
    - image: quay.io/buildah/stable
      name: build
      resources: {}
      script: |
        buildah \
        bud \
        --format \
        docker \
        --tls-verify=$(inputs.params.TLSVERIFY) \
        --storage-driver=vfs \
        --layers \
        -f \
        /gen-source/Dockerfile \
        -t \
        $(cat $(results.image-url.path)) \
        .
      securityContext:
        privileged: true
      volumeMounts:
        - mountPath: /var/lib/containers
          name: varlibcontainers
        - mountPath: /gen-source
          name: gen-source
      workingDir: /gen-source
    - image: quay.io/buildah/stable
      name: push
      resources: {}
      script: |
        #!/bin/bash

        IMAGE_URL=$(cat $(results.image-url.path))
        REG_CRED=$(cat $(results.registry-cred.path) | base64 -d)
        if [ "$REG_CRED" != "" ]; then
            CRED="--creds=$REG_CRED"
        fi

        buildah \
        push \
        --tls-verify=$(inputs.params.TLSVERIFY) \
        --storage-driver=vfs \
        $CRED \
        $IMAGE_URL \
        docker://$IMAGE_URL
      securityContext:
        privileged: true
      volumeMounts:
        - mountPath: /var/lib/containers
          name: varlibcontainers
  volumes:
    - emptyDir: {}
      name: varlibcontainers
    - emptyDir: {}
      name: gen-source
  workspaces:
    - description: Git source directory
      name: git-source
