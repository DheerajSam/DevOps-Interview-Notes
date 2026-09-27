1. Explain the CI/CD pipeline.

Answer:

When I push code to the main branch, GitHub Actions starts the pipeline. It checks out the code, sets up Node.js, installs dependencies, runs Jest tests, and performs an application health check. Then it builds the Docker image, scans the image using Trivy for HIGH and CRITICAL vulnerabilities, and only after the scan passes, pushes the image to DockerHub. The image is tagged with both the Git commit SHA and latest.

2. Why did you use multi-stage Docker builds?

Answer:

I used a multi-stage Docker build to separate the dependency/build stage from the runtime stage. The final image contains only what is required to run the application, which reduces unnecessary files and packages in the runtime container.

3. Why do you scan the Docker image before pushing it?

Answer:

I don't want a vulnerable image to be published to DockerHub. Trivy scans the exact Docker image that was built by the pipeline, and if it finds HIGH or CRITICAL vulnerabilities, the pipeline stops before the push step.

4. Why use the Git commit SHA as an image tag?

Answer:

The commit SHA gives every image a unique and traceable version. If I have an issue with a deployment, I can identify exactly which source-code commit produced that image.

Example:

username/devops-fullstack-app:9180f4...
5. Why also use the latest tag?

Answer:

latest provides a convenient reference to the most recently published image. For traceability, I prefer the SHA tag when identifying a specific release.

6. How does the health check work?

Answer:

During CI, I start the Node.js application in the background and call the /health endpoint using curl. If the endpoint doesn't return successfully, the pipeline fails. I also use a cleanup trap so the application process is terminated when the step finishes.

7. Why does the Docker container run as the node user?

Answer:

I don't need root privileges to run the Node.js application. Running as the non-root node user follows the principle of least privilege and reduces the impact if the application is compromised.

8. Why did you remove npm from the runtime image?

Answer:

npm is needed during dependency installation, but the running application doesn't need npm. I removed it from the final runtime image to reduce unnecessary packages and also reduce the container's vulnerability surface.

9. How does Prometheus collect metrics from your application?

Answer:

I use the Prometheus client library in the Node.js application. The application exposes metrics through /metrics. Prometheus can scrape that endpoint periodically and collect metrics such as HTTP request count and request duration.

10. What happens if Trivy finds a HIGH vulnerability?

Answer:

The Trivy step exits with a non-zero status because the pipeline is configured with exit-code: 1 for HIGH and CRITICAL vulnerabilities. GitHub Actions therefore stops the job and the Docker image isn't pushed.

11. Why is Kubernetes deployment currently disabled?

Answer:

The Kubernetes environment was created as a hands-on AWS environment, but I'm not keeping the EC2 and Kubernetes infrastructure running continuously because of cost. The Kubernetes manifests and deployment workflow are retained so the environment can be recreated and deployment can be tested again.

This is a good answer because it is completely honest about the current state.

12. How would you deploy this application to Kubernetes?

Answer:

I would first make sure the Kubernetes cluster is available, then apply the Deployment and Service manifests. The Deployment would create the required replicas, and the Service would provide network access to the application. The image would be referenced using the specific Git commit SHA rather than relying only on latest.

13. How would you troubleshoot a failed CI pipeline?

Answer:

First I would identify which stage failed. For example, if tests fail, I would check the application/test output. If the Docker build fails, I would inspect the Docker build logs. If Trivy fails, I would check the vulnerability report and identify the affected package or base image. If DockerHub push fails, I would check authentication and repository permissions.

14. Why did you add GitHub Actions caching?

Answer:

Docker builds can repeatedly download the same layers. Buildx caching allows GitHub Actions to reuse previous Docker build layers, which reduces build time when there haven't been significant changes to those layers.

15. Why use both tests and a health check?

Answer:

They validate different things. Jest tests verify application behavior at the code level. The health check verifies that the application can actually start and respond over HTTP.

16. What is the difference between /health and /metrics?

Answer:

/health is mainly used to determine whether the application is running and responding correctly. /metrics exposes monitoring data in Prometheus format, such as request counts and request duration.

17. Why use Kubernetes Deployment instead of creating individual Pods?

Answer:

A Deployment manages the desired number of replicas and handles updates and replacement of Pods. If a Pod fails, Kubernetes can create another one to maintain the desired state. It also supports rolling updates.

18. What is a Kubernetes Service?

Answer:

A Service provides stable network access to a group of Pods. Pods can be recreated and their IP addresses can change, but clients can continue communicating through the Service.

19. What is Terraform doing in this project?

Answer:

Terraform is used as Infrastructure as Code to define the AWS infrastructure required for the environment, such as the VPC, networking components, security groups, IAM configuration and EC2 resources. Instead of manually creating everything through the AWS console, the infrastructure can be defined and recreated from configuration.

20. If your interviewer asks, "Is this production?"

Answer:

It's a hands-on DevOps project designed around a production-style architecture. I built and tested the CI/CD, Docker, security scanning, infrastructure, Kubernetes and observability components. The AWS/Kubernetes environment isn't kept running continuously, but the configuration is maintained so it can be recreated.