# Live Demo

The demo comprises of two sections: Monitoring and Infrastructure-as-Code (IaC) using Ansible.
For the monitoring section, please prepare the following tools:

- Docker
- Docker Compose

To begin, run all containers:

```shell
docker compose up --detach
# In some Docker & Docker Compose installation, the command is: docker-compose up --detach
```

As for the IAC demo, the demo requires Python 3 and Ansible.
Since we will configure an Ubuntu server , Ansible needs to run on a non-Windows environment.
Therefore, you need to run in on either Mac or Linux.
If you are using Windows, then you can run the example from WSL environment.

## Demo: Monitoring

1. Open the Grafana dashboard at [`http://localhost:3000/dashboard`](http:/localhost:3000/dashboards).
2. Choose **Spring Petclinic Metrics** dashboard to open it.
3. You will see 7 widgets in the dashboard.
   Each widget displays visualisation based on a query written in PromQL to the Prometheus data source that is running within the private Docker network (i.e., `prometheus-server` running on port `9090` in the internal network created by `docker compose`).
4. Let's try looking at one widget and the corresponding PromQL query example.
   Click on the **HTTP Request Activity** widget title and choose **Edit** button.
5. You will see a bigger widget view, including the query builder panel at the bottom of the view.
6. There are two existing queries in the query builder. The first query is `sum(rate(http_server_requests_seconds_count[1m]))`. It comprises of several parts, such as:
    - `http_server_requests_seconds_count` --> a _counter metric_ in Prometheus. It represents the total count of HTTP requests your server gets. The number increases over time each time new requests come in.
    - `[1m]` --> a _time window_ or _range vector_. It specifies that we are interested in data or changes in the last 1 minute.
    - `rate()` --> a function in PromQL that calculate the _per-second average rate_ of time series in a range vector.
    - `sum()` --> a function in PromQL that adds up all the values produced by the wrapped function.
7. The latter query, i.e., `sum(rate(http_server_requests_seconds_count{status=~"5.."}[1m]))`, is similar. However, it adds a condition for filtering metrics based on label values. See the `{status=~"5.."}` part in the query? It means the query only take metrics where the `status` label has matching any HTTP status codes that start with `5` (e.g., `500`, `502`, etc.).

### Simulate Load Test

Go back to the main dashboard view and keep the browser window open.
Let's try to simulate a load test and see how the Grafana visualises the activities.

The project includes a JMeter test plan that can be found inside the [`spring-petclinic-api-gateway` test code](./spring-petclinic-api-gateway/src/test/jmeter).
The test plan requires two JMeter plugins: Plugins Manager and Custom Thread Groups.
First, install the Plugins Manager plugin by following the instructions [here (click)](https://jmeter-plugins.org/wiki/PluginsManager/).
Once it has been installed, open JMeter GUI and go to the Plugins Manager menu.
From there, you can install the Custom Thread Groups plugin.

Run the test plan by loading it to the JMeter GUI or via `jmeter` command.
Let the test plan run for a while. It will take five minutes to finish.
While waiting, you can see the Grafana dashboard is being updated.

### Spring Boot Actuator & Custom Metrics

The example above shows that we can capture metrics related to HTTP traffic. It is also, coincidentally, some of the default metrics that are exported when we enable Spring Boot Actuator component in a Spring Boot-based project.

While the default metrics are useful, we may also want to collect business-related metrics from the application.
For example, how many pets have been registered by pet owners?

Let's look at `PetResource.java` in `spring-petclinic-customers-service` directory:

```java
// Omitted for brevity ...
@RestController
@Timed("petclinic.pet")
@RequiredArgsConstructor
@Slf4j
class PetResource {

    private final PetRepository petRepository;
    private final OwnerRepository ownerRepository;


    @GetMapping("/petTypes")
    public List<PetType> getPetTypes() {
        return petRepository.findPetTypes();
    }

    @PostMapping("/owners/{ownerId}/pets")
    @ResponseStatus(HttpStatus.CREATED)
    public Pet processCreationForm(
        @RequestBody PetRequest petRequest,
        @PathVariable("ownerId") @Min(1) int ownerId) {

        Owner owner = ownerRepository.findById(ownerId)
            .orElseThrow(() -> new ResourceNotFoundException("Owner " + ownerId + " not found"));

        final Pet pet = new Pet();
        owner.addPet(pet);
        return save(pet, petRequest);
    }
    // Omitted for brevity ...
}
```

`@Timed` annotation adds instrumentation on each methods in the applied class.
In the example above, the `GET` and `POST` request handler invocations will be measured.
The collected measurements will be available in `petclinic.pet` metrics namespace, which in turn, collected by Prometheus.
Then, we can query the metrics corresponding to the number of created pet by using the following PromQL query: `petclinic_pet_seconds_count{method="processCreationForm", exception="none"}`.

## Demo: Infrastructure-as-Code

Let's look at the instructions on [how to install Docker (click)](https://docs.docker.com/engine/install/ubuntu/) on Ubuntu Linux,
specifically in the ["Install using the `apt` repository" section](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository).
There are seven lines of commands to run in order to set up the package repository for installing Docker,
as outlined in the following code snippet (taken from the official documentation):

```shell
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Then, it is followed by the actual commands to install Docker:

```shell
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

It is a common procedure for installing Docker on Linux
and works fine if we are installing tool to a single machine.
However, what if we want to install Docker on _multiple machines_?
Sure, developers can execute the same procedure all over again in each machine.
But it will become repetitive and prone to error since it is not automated.

This is where deployment automation tool (or, often also called as Infrastructure-as-Code) such as Ansible can help.
Instead of manually executing the commands for setting up the infrastructure and the environments in it,
Ansible lets developers to define the procedures (also known as: _playbook_) that will be executed on a set of target machines (also known as: _inventory_).

### Setting Up Ansible

To run Ansible, you need to install the required dependencies listed in `requirements.txt`:

```shell
pip install -r requirements.txt
```

> Note: If you are using Windows, please run the example on WSL or a Linux VM!

Currently, it only contains a single package: `ansible-dev-tools`.
It provides not only the `ansible` command, but also several other tools that can assist writing Ansible playbooks (e.g., `ansible-lint`).

Next, install the required Ansible _Roles_ and _Collections_.
Both provides a set of Ansible _Modules_ that can be used to run a collection of tasks encapsulated in the module.
The Roles and Collections are listed in `requirements.yml`:

```shell
ansible-galaxy install collection -r requirements.yml
ansible-galaxy install role -r requirements.yml
```

> Note: Role and Collection are similar in Ansible. Both are containers for a set of modules,
> and each module comprises of a collection of tasks. Traditionally, Ansible only provide Role.
> But in more recent Ansible version, they introduced another type of module container named Collection.
> A Collection can contain a collection of Roles.

### Ansible Inventory

Now, we define the machines where the Ansible will run the modules.
Create a new file named `hosts.yml` (or, you can check a [provided example](./hosts.yml); make sure you change the `ansible_host` to the address of your own machine, either domain or IP address):

```yaml
---
demo:
  hosts:
    advprog1:
      ansible_host: <MACHINE_1_IP_OR_DOMAIN>
      ansible_port: 22
      ansible_user: root
```

> Note: Currently, it only contains one machine for demonstration purpose.
> If you want to prepare more than one machine,
> you can create multiple VMs locally or on a cloud service provider.

Make sure the target machines have Python installed.
Ansible will use the available Python runtime on the target machines to run the tasks.
To do a simple check, run a _ping_ command using `ansible`:

```shell
ansible demo -i hosts.yml -m ping
```

Ansible will attempt to connect to each machine defined under `demo` group using `root` user.
Then, it will invoke the `ping` command.
The resulting output would be similar to:

```shell
10.1.149.232 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Ansible Playbook

Let's look at the [example playbook](./playbook_demo-advprog.yml):

```yaml
  # A playbook that improves (hardens) the machine using `devsec.hardening` collection and roles inside it,
  # thus making the server more secure than default installation.
- name: Harden server
  hosts: demo
  collections:
    - devsec.hardening
  roles:
    - devsec.hardening.os_hardening
    - devsec.hardening.ssh_hardening
  vars:
    sysctl_overwrite:
      # Enable IPv4 traffic forwarding
      net.ipv4.ip_forward: 1
    os_auth_pw_max_age: 99999
    sftp_enabled: true
    ssh_server_ports:
      - "22"
    ssh_permit_root_login: "yes"

  # A playbook for installing Docker using `geerlingguy.docker` role.
- name: Install Docker
  hosts: demo
  roles:
    - geerlingguy.docker
  vars:
    docker_edition: "ce"
    docker_compose_version: "1.28.4"

- name: Install node_exporter
  hosts: demo
  roles:
    - geerlingguy.node_exporter
```

To run the playbook:

```shell
ansible-playbook -i hosts.yml playbook_demo-advprog.yml
```

## Possible Issues

- If you cannot start containers due to conflicting ports,
  make sure all the published ports are not used by other running processes on the host machine.
  You can change the published ports by modifying the `ports` section of a service in [`docker-compose.yml`](./docker-compose.yml) file.
- The services sometime unable to connect to Zipkin, thus dropping all spans.
  The cause is still unknown. The distributed tracing worked when tested on a laptop, but got broken when tested on a desktop PC.
- If Ansible cannot make connection to a target machine,
  make sure the address is accessible from the host machine.
  For example, if your machine is located on GCP, make sure the IP address you used is the public IP.
  Furthermore, make sure the firewall allows incoming (ingress) traffic at port 22.
