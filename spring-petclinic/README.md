1. General agent Vs 2. Specific agent
    - General agent is the worker infrastructure it could be a VM, physical server, or a container
    - Specific agent is a worker that is designed to perform a specific task on that infrastructure.
    ex. agent any, agent {docker{image 'aquasec/trivy'}} -> means run this docker container on any available node

