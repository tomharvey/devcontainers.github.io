---
layout: implementors
title:  "Development Container Specification"
shortTitle: "Specification"
author: Microsoft
index: 1
---

The purpose of the **development container** specification is to provide a way to enrich containers with the content and metadata necessary to enable development inside them. These container **environments** should be easy to use, create, and recreate. 

A **development container** is a container in which a user can develop an application.  Tools that want to implement this specification should provide a set of features/commands that give more flexibility to users and allow **development containers** to scale to large development groups.

An **environment** is defined as a logical instance of one or more **development containers**, along with any needed side-car containers. An environment is based on one set of metadata that can be managed as a single unit. Users can create multiple **environments** from the same configuration metadata for different purposes.

# <a href="#metadata" name="metadata" class="anchor"> Metadata </a>

**Development containers** allow one to define a repeatable development environment for a user or team of developers that includes the execution environment the application needs. A development container defines an environment in which you develop your application before you are ready to deploy. While deployment and development containers may resemble one another, you may not want to include tools in a deployment image that you use during development and you may need to use different secrets or other settings. 

Furthermore, working inside a development container can require additional **metadata** to drive tooling or service experiences than you would normally need with a production container. Providing a structured and consistent form for this metadata is a core part of this specification.

A **development container** is composed of a definition (e.g. contained in a `devcontainer.json` file) that deterministically creates containers under the control of the user.

## <a href="#devcontainerjson" name="devcontainerjson" class="anchor"> `devcontainer.json` </a>

While the structure of this metadata is critical, it is also important to call out how this data can be represented on disk where appropriate. While other representations may be added over time, metadata can be stored in a JSON with Comments file called `devcontainer.json` today. Products using it should expect to find a devcontainer.json file in one or more of the following locations (in order of precedence):

- .devcontainer/devcontainer.json
- .devcontainer.json
- .devcontainer/<folder>/devcontainer.json (where <folder> is a sub-folder, one level deep)

It is valid that these files may exist in more than one location, so consider providing a mechanism for users to select one when appropriate.

## <a href="#image-metadata" name="image-metadata" class="anchor"> Image Metadata </a>

Certain dev container metadata properties can be stored in an image label as an array of metadata snippets. This allows them to be stored in prebuilt images, such that, the image and its related configuration are self-contained. These contents should then be merged with any local devcontainer.json file contents at the time the container is created. An array is used so subsequent image builds can simply append changes to the array rather than attempting to merge at that point - which improves compatibility with arbitrary image build systems.

Metadata should be representative of with the following structure, using one entry per [Dev Container Feature](../features) and devcontainer.json (see table below for the full list):

```json
[
	{
		"id"?: string,
		"init"?: boolean,
		"privileged"?: boolean,
		"capAdd"?: string[],
		"securityOpt"?: string[],
		"entrypoint"?: string,
		"mounts"?: [],
		...
		"customizations"?: {
			...
		}
	},
	...
]
```

To simplify adding this metadata for other tools, we also support having a single top-level object with the same properties.

The metadata is added to the image as a `devcontainer.metadata` label with a JSON string value representing the above array or single object.

### <a href="#merge-logic" name="merge-logic" class="anchor"> Merge Logic </a>

To apply the metadata together with a user's devcontainer.json at runtime, the following merge logic by property is used. The table also notes which properties are currently supported coming from the devcontainer.json and from the Feature metadata- this will change over time as we add more properties.

| Property | Type/Format | Merge Logic | devcontainer.json | devcontainer-feature.json |
| -------- | ----------- | ----------- | :---------------: | :--------------: |
| `id` | E.g., `ghcr.io/devcontainers/features/node:1` | Not merged. |   | x |
| `init` | `boolean` | `true` if at least one is `true`, `false` otherwise. | x | x |
| `privileged` | `boolean` | `true` if at least one is `true`, `false` otherwise. | x | x |
| `capAdd` | `string[]` | Union of all `capAdd` arrays without duplicates. | x | x |
| `securityOpt` | `string[]` | Union of all `securityOpt` arrays without duplicates. | x | x |
| `entrypoint` | `string` | Collected list of all entrypoints. |   | x |
| `mounts` | `(string \| { type, src, dst })[]` | Collected list of all mountpoints. Conflicts: Last source wins. | x | x |
| `onCreateCommand` | `string \| string[] \| {[key: string]: string \| string[]}` | Collected list of all onCreateCommands. | x | x  |
| `updateContentCommand` | `string \| string[] \| {[key: string]: string \| string[]}` | Collected list of all updateContentCommands. | x | x  |
| `postCreateCommand` | `string \| string[] \| {[key: string]: string \| string[]}` | Collected list of all postCreateCommands. | x |  x |
| `postStartCommand` | `string \| string[] \| {[key: string]: string \| string[]}` | Collected list of all postStartCommands. | x |  x |
| `postAttachCommand` | `string \| string[] \| {[key: string]: string \| string[]}` | Collected list of all postAttachCommands. | x | x  |
| `waitFor` | enum | Last value wins. | x |   |
| `customizations` | Object of tool-specific customizations. | Merging is left to the tools. | x | x |
| `containerUser` | `string` | Last value wins. | x |   |
| `remoteUser` | `string` | Last value wins. | x |   |
| `userEnvProbe` | `string` (enum) | Last value wins. | x |   |
| `remoteEnv` | Object of strings. | Per variable, last value wins. | x |   |
| `containerEnv` | Object of strings. | Per variable, last value wins. | x |   |
| `overrideCommand` | `boolean` | Last value wins. | x |   |
| `portsAttributes` | Map of ports to attributes. | Per port (not per port attribute), last value wins. | x |   |
| `otherPortsAttributes` | Port attributes. | Last value wins (not per port attribute). | x |   |
| `forwardPorts` | `(number \| string)[]` | Union of all ports without duplicates. Last one wins (when mapping changes). | x |   |
| `shutdownAction` | `string` (enum) | Last value wins. | x |   |
| `updateRemoteUserUID` | `boolean` | Last value wins. | x |   |
| `hostRequirements` | `cpus`, `memory`, `storage` | Max value wins. | x |   |
{: .table .table-bordered .table-responsive}

Variables in string values will be substituted at the time the value is applied. When the order matters, the devcontainer.json is considered last.

### <a href="#metadata-notes" name="metadata-notes" class="anchor"> Notes </a>

- Passing the label as a `LABEL` instruction in the Dockerfile:
	- The size limit on Dockerfiles is around 1.3MB. The line length is limited to 65k characters.
	- Using one line per feature should allow for making full use of these limits.
- Passing the label as a command line argument:
	- There is no size limit documented for labels, but the daemon returns an error when the request header is >500kb.
	- The 500kb limit is shared, so we cannot use a second label in the same build to avoid it.
	- If/when this becomes an issue we could embed the metadata as a file in the image (e.g., with a label indicating it).

## <a href="#orchestration-options" name="orchestration-options" class="anchor"> Orchestration options </a>

A core principle of this specification is to seek to enrich existing container orchestrator formats with development container metadata where appropriate rather than replacing them. As a result, the metadata schema includes a set of optional properties for interoperating with different orchestrators. Today, the specification includes scenario-specific properties for working without a container orchestrator (by directly referencing an image or Dockerfile) and for using Docker Compose as a simple multi-container orchestrator. At the same time, this specification leaves space for further development and implementation of other orchestrator mechanisms and file formats. 

The following section describes the differences between those that are supported now. 

## <a href="#image-based" name="image-based" class="anchor"> Image based </a>

Image based configurations only reference an image that should be reachable and downloadable through `docker pull` commands. Logins and tokens required for these operations are execution environment specific. The only required parameter is `image`. The details are [here](../json_reference#image-specific).

## <a href="#dockerfile-based" name="dockerfile-based" class="anchor"> Dockerfile based </a>

These configurations are defined as using a `Dockerfile` to define the starting point of the **development containers**. As with image based configurations, it is assumed that any base images are already reachable by **Docker** when performing a `docker build` command. The only required parameter in this case is the relative reference to the `Dockerfile` in `build.dockerfile`. The details are [here](../json_reference#image-specific).

There are multiple properties that allow users to control how `docker build` works:

- `build.context` 
- `build.args`
- `build.target`
- `build.cacheFrom`

## <a href="#docker-compose-based" name="docker-compose-based" class="anchor"> Docker Compose based </a>

Docker Compose configurations use `docker-compose` (which may be Docker Compose V1 or aliased Docker Compose V2) to create and manage a set of containers required for an application. As with the other configurations, any images required for this operation are assumed to be reachable. The required parameters are:

- `dockerComposeFile`: the reference to the Docker Compose file(s) to be used.
- `service`: declares the **main** container that will be used for all other operations. Tools are assumed to also use this parameter to connect to the **development container**, although they can provide facilities to connect to the other containers as required by the user.
- `runServices`: an optional property that indicates the set of services in the `docker-compose` configuration that should be started or stopped with the environment.

It is important to note that **image** and **dockerfile** properties are not needed since Docker Compose supports them natively in the format. 

# <a href="#other-options" name="other-options" class="anchor"> Other options </a>

In addition to the configuration options explained above, there are other settings that apply when creating **development containers** to facilitate their use by developers. 

A complete list of available metadata properties and their purposes can be found in the [`devcontainer.json` reference](https://aka.ms/devcontainer.json). However, we will describe the critical ones below in more detail.

## <a href="#features" name="features" class="anchor"> Features</a>

Development container "Features" are self-contained, shareable units of installation code and development container configuration. The name comes from the idea that referencing one of them allows you to quickly and easily add more tooling, runtime, or library "features" into your development container for you or your collaborators to use.

They are applied to container images as a secondary build step and can affect a number of dev container configuration settings. See the [features documentation](../features) for more details.

## <a href="#environment-variables" name="environment-variables" class="anchor"> Environment variables </a>

Environment variables can be set at different points in the dev container lifecycle. With this in mind, **development containers** support two classes of environment variables:

* **Container**: These variables are part of the container when it is created and are available at all points in its lifecycle. This concept is native to containers and can be set in the container image itself, using `containerEnv` for **image** and **Dockerfile** scenarios or using orchestrator specific properties like `env` in **Docker Compose** files.
* **Remote**: These variables should be set by a **development container** supporting tool as part of configuring its runtime environment. Users can set these using the `remoteEnv` property and implementing tools or services may add their own for specific scenarios (e.g., secrets). These variables can change during the lifetime of the container, and are added after the container's `ENTRYPOINT` has fired.

The reason for this separation is it allows for the use of information not available at image build time and simplifies updating the environment for project/repository specific needs without modifying an image. With this in in mind, it's important to note that implementing tools should also support the [dynamic variable syntax](../json_reference/#variables-in-devcontainerjson) described in the metadata reference document.

Another notable and important environment variable related property is **`userEnvProbe`**. Implementing tools should use this property to "probe" for expected environment variables using the specified type of shell. However, it does not specify that this type of shell needs to be used for all sub-processes (given the performance impact). Instead, "probed" environment variables should be merged with Remote environment variables for any processes the implementer injects after the container is created.  This allows implementors to emulate developer expected behaviors around values added to their profile and rc files. 

## <a href="#mounts" name="mounts" class="anchor"> Mounts </a>

Mounts allow containers to have access to the underlying machine, share data between containers and to persist information between **development containers**. 

A default mount should be included so that the source code is accessible from inside the container. Source code is stored outside of the container so that a developer's in-flight edits can be extracted, or a new container created in the event a container no longer starts.

## <a href="#workspace-folder" name="workspace-folder" class="anchor"> workspaceFolder and workspaceMount </a>

The default mount point for the source code can be set with the `workspaceMount` property for **image** and **dockerfile** scenarios or using the built in `mounts` property in **Docker Compose** files. This folder should point to the root of a repository (where the `.git` folder is found) so that source control operations work correctly inside the container.

The `workspaceFolder` can then be set to the default folder inside the container that should used in the container. Typically this is either the mount point in the container, or a sub-folder under it. Allowing a sub-folder to be used is particularly important for monorepos given you need the `.git` folder to interact with source control but developers are typically are interacting with a specific sub-project within the overall repository. 

See [`workspaceMount` and `workspaceFolder`](../json_reference#image-specific) for reference.

## <a href="#users" name="users" class="anchor"> Users </a>

Users control the permissions of applications executed in the containers, allowing the developer to control them. The specification takes into account two types of user definitions:

* **Container User**: The user that will be used for all operations that run inside a container. This concept is native to containers. It may be set in the container image, using the `containerUser` property for  **image** and **dockerfile** scenarios, or using an orchestratric specific property like `user` property in Docker Compose files.
* **Remote User**: Used to run the [lifecycle](#lifecycle) scripts inside the container. This is also the user tools and editors that connect to the container should use to run their processes. This concept is not native to containers. Set using the `remoteEnv` property in all cases and defaults to the container user.

This separation allows the `ENTRYPOINT` for the image to execute with different permissions than the developer and allows for developers to switch users without recreating their containers.

# <a href="#lifecycle" name="lifecycle" class="anchor"> Lifecycle </a>

A development environment goes through different lifecycle events during its use in the outer and inner loop of development.

- Configuration Validation
- Environment Creation
- Environment Stop
- Environment Resume

## <a href="#configuration-validation" name="configuration-validation" class="anchor"> Configuration Validation </a>

The exact steps required to validate configuration can vary based on exactly where the **development container** metadata is persisted. However, when considering a `devcontainer.json` file, the following validation should occur:

1. Validate that a workspace source folder has been provided. It is up to the implementing tool to determine what to do if no source folder is provided.
2. Search for a `devcontainer.json` file in one of the locations [above](#devcontainerjson) in the workspace source folder. 
3. If no `devcontainer.json` is found, it is up to the implementing tool or service to determine what to do. This specification does not dictate this behavior.
4. Validate that the metadata (for example `devcontainer.json`) contains all parameters required for the selected configuration type.

## <a href="#environment-creation" name="environment-creation" class="anchor"> Environment Creation </a>

The creation process goes through the steps necessary to go from the user configuration to a working **environment** that is ready to be used.

### <a href="#initialization" name="initialization" class="anchor"> Initialization </a> 

During this step, the following is executed:
- Validate access to the container orchestrator specified by the configuration.
- Execution of `initializeCommand`.

### <a href="#image-creation" name="image-creation" class="anchor"> Image Creation </a> 

The first part of environment creation is generating the final image(s) that the **development containers** are going to use. This step is orchestrator dependent and can consist of just pulling a Docker image, running Docker build, or docker-compose build. Additionally, this step is useful on its own since it permits the creation of intermediate images that can be uploaded and used by other users, thus cutting down on creation time. It is encouraged that tools implementing this specification give access to a command that just executes this step.

This step executes the following tasks:

1. [Configuration Validation](#configuration-validation) 
2. Pull/build/execute of the defined container orchestration format to create images.
3. Validate the result of these operations.

### <a href="#container-creation" name="container-creation" class="anchor"> Container Creation </a> 

After image creation, containers are created based on that image and setup.

This step executes the following tasks:

1. [Optional] Perform any required user UID/GID sync'ing (more next)
2. Create the container(s) based on the properties specified above.
3. Validate the container(s) were created successfully.

Note that container [mounts](#mounts), [environment variables](#environment-variables), and [user](#users) configuration should be applied at this point. However, remote user and environment variable configuration should **not** be.

UID/GID sync'ing is an optional task for Linux (only) and that executes if the `updateRemoteUserUID` property is set to true and a `containerUser` or `remoteUser` is specified. In this case, an image update should be made prior to creating the container to set the specified user's UID and GID to match the current local user’s UID/GID to avoid permission problems with bind mounts. Implementations **may** skip this task if they do not use bind mounts on Linux, or use a container engine that does this translation automatically.

### <a href="#post-container-creation" name="post-container-creation" class="anchor"> Post Container Creation </a>

At the end of the container creation step, a set of commands are executed inside the **main** container: 
- `onCreateCommand`, `updateContentCommand` and `postCreateCommand`. This set of commands is executed in sequence on a container the first time it's created and depending on the creation parameters received. You can learn more in the [documentation on lifecycle scripts](../json_reference/#lifecycle-scripts). By default, `postCreateCommand` is executed in the background after reporting the successful creation of the development environment.
- If the `waitFor` property is defined, then execution should block until all commands in the sequence up to the specified property have executed. This property defaults to `updateContentCommand`.

Remote [environment variables](#environment-variables) and [user](#users) configuration should be applied to all created processes in the container (inclusive of `userEnvProbe`).

### <a href="#implementation-specific-steps" name="implementation-specific-steps" class="anchor"> Implementation specific steps </a>

After these steps have been executed, any implementation specific commands can safely execute. Specifically, any processes required by the implementation to support other properties in this specification should be started at this point. These may occur in parallel to any non-blocking, background post-container creation commands (as dictated by the `waitFor` property).

Any user facing processes should have remote [environment variables](#environment-variables) and [user](#users) configuration applied (inclusive of `userEnvProbe`).

For example, in the [CLI reference implementation](https://github.com/devcontainers/cli), this is the point in which anything executed with `devcontainer exec` would run.

Typically, this is also the step where implementors would apply config or settings from the `customizations` section of the dev container metadata (e.g., VS Code installs extensions based on the `customizations.vscode.extensions` property). Examples of these can be found in the [supporting tools section](../../supporting) reference. However, applying these at this point is not strictly required or mandated by this specification.

Once these final steps have occurred, implementing tools or services may connect to the environment as they see fit.

## <a href="#environment-stop" name="environment-stop" class="anchor"> Environment Stop </a>

The intention of this step is to ensure all containers are stopped correctly based on the appropriate orchestrator specific steps to ensure no data is lost. It is up to the implementing tool or service to determine when this event should happen.

## <a href="#environment-resume" name="environment-resume" class="anchor"> Environment Resume </a>

While it is not a strict requirement to keep a **development container** after it has been stopped, this is the most common scenario.

To resume the environment from a stopped state:
1. Restart all related containers.
2. Follow the appropriate [implementation specific steps](#implementation-specific-steps).
3. Additionally, execute the `postStartCommand` and `postAttachCommand` in the container.

Like during the create process, remote [environment variables](#environment-variables) and [user](#users) configuration should be applied to all created processes in the container (inclusive of `userEnvProbe`).

## <a href="#parallel-exec" name="parallel-exec" class="anchor"> Parallel lifecycle script execution </a>

Dev containers support a single command for each of its lifecycle scripts. While serial execution of multiple commands can be achieved with `;`, `&&`, etc., parallel execution deserves first-class support.

All lifecycle scripts have been extended to support `object` types. The key of the `object` will be a unique name for the command, and the value will be the `string` or `array` command. Each command must exit successfully for the stage to be considered successful.

Each entry in the `object` will be run in parallel during that lifecycle step.

### Example

```json
{
  "postCreateCommand": {
    "server": "npm start",
    "db": ["mysql", "-u", "root", "-p", "my database"]
  }
}
```

# <a href="#definitions" name="definitions" class="anchor"> Definitions </a>
#### <a href="#project-workspace-folder" name="project-workspace-folder" class="anchor"> Project Workspace Folder </a>

The **project workspace folder** is where an implementing tool should begin to search for `devcontainer.json` files. If the target project on disk is using git, the **project workspace folder** is typically the root of the git repository. 
