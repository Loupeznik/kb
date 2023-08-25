# General
## Find out if I am inside a Docker container
**PHP example**
```php
public function __construct()
{
  // if I am inside a Docker container, use other path than if I am on a real system
  if (file_exists('/.dockerenv'))
  {
    $this->env = Yaml::parseFile($_SERVER["DOCUMENT_ROOT"] . '/.env.yml');
  }
  else 
  {
      $this->env = Yaml::parseFile(__DIR__ . '/../../.env.yml'); 
  }
}
```
**Bash example**
```shell
#!/bin/bash
if [ -f /.dockerenv ]; then
    echo "I'm inside a container ;(";
else
    echo "I'm living in real world!";
fi
```

## Update container restart policy
```shell
CONTAINER_NAME="my_container"
sudo docker update --restart=always $CONTAINER_NAME
```

## Access host services
Use `host.docker.internal` or default gateway of the Docker interface
