# Docker in Rootless Mode

## Docker runs as Root?

The Docker daemon runs as root by default. If you add users to the Docker group, you've essentially given them root on the system. This also means that container escapes can be especially dangerous. SELinux and AppArmor can help protect your system, but it still not entirely secure. 

As of Docker 19.03.0, there is now an *Experimental* feature to run the Docker daemon as a non-root user. 

