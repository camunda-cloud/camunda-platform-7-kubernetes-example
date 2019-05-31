---


---

<h1 id="todos">TODOS</h1>
<ul>
<li>Update documentation to point to github</li>
<li>send to josh for review</li>
<li>credit lars</li>
<li>Publish docker image and edit location to latest</li>
</ul>
<p>Are you running Kubernetes now? Ready to move your Camunda BPM instances off of VMs, or just try it out on Kubernetes? We will address some common configurations and provide some building blocks you can tailor to your particular needs. A working knowledge of Kubernetes is assumed.</p>
<h2 id="prerequisites">Prerequisites</h2>
<ul>
<li>A working <a href="https://kubernetes.io/">Kubernetes</a> cluster
<ul>
<li><a href="https://cloud.google.com/free/">GKE</a> or minikube are a good way to get started</li>
</ul>
</li>
<li>[Optional] <a href="https://github.com/kubernetes-sigs/kustomize">Kustomize &gt; 2.0.3</a> for managing yaml overlays without forking the whole manifest, allowing you to <code>git pull --rebase</code> future improvements
<ul>
<li>Variable support in the ingress was added after 2.0.3 was released, so for now and make sure that <a href="https://gist.github.com/afirth/fabc04406eb584601b473f599eb0170a">go installed binaries are available on your PATH</a> and <code>go get sigs.k8s.io/kustomize</code></li>
</ul>
</li>
<li>[Optional] Skaffold for building your own docker images
<ul>
<li>download the latest release with <code>curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64 &amp;&amp; chmod +x skaffold &amp;&amp; sudo mv skaffold /usr/local/bin</code></li>
</ul>
</li>
</ul>
<h1 id="what-is-camunda-bpm">What is Camunda BPM</h1>
<p><a href="https://camunda.com">Camunda BPM</a> is an <a href="https://github.com/camunda/camunda-bpm-platform">open source</a> platform for workflow and decision automation that brings business users and software developers together. It is ideal for coordinating and connecting humans, (micro)services, or even robots! You can read more about some use cases <a href="https://camunda.com/solutions/">here</a>.</p>
<h1 id="why-run-on-kubernetes">Why run on Kubernetes</h1>
<p>Kubernetes has become the de facto standard for running modern applications on Linux. By using system calls instead of emulating the hardware layer, and allowing the kernel to manage memory and task switching, overhead and startup times are minimized. However, the greatest benefit may come in the standard API it provides for configuring the infrastructure all applications need, such as storage, networking, and monitoring. 5 years old in June 2019, it’s arguably the second largest open source project ever (behind Linux), and is stabilizing in feature set after the rapid iteration of the past few years as it becomes critical to production workloads around the globe.</p>
<p>Camunda BPM Engine can easily connect easily to other applications running in the same cluster, and Kubernetes provides great scalability, allowing you to grow your infrastructure costs only when the software really needs it (and shrink it moments later).</p>
<p>Visibility is also greatly improved by tools like Prometheus, Grafana, Loki, Fluentd and Elasticsearch allowing a centralized view of all cluster workloads. We’ll look at how to inject a Prometheus exporter into the JVM today.</p>
<h1 id="tasks">Tasks</h1>
<p>There are three areas where we need to do some work to the Camunda BPM docker image (<a href="https://github.com/camunda/docker-camunda-bpm-platform">github</a>) to become more “cloud native”.</p>
<ol>
<li>Logs and metrics</li>
<li>Database connections</li>
<li>Authentication</li>
</ol>
<p>Today we will go through some techniques to address these, and show a workflow that might work for you.</p>
<h2 id="logs-and-metrics">Logs and Metrics</h2>
<p>In kube land, Prometheus has become the standard for capturing metrics. It fills the same niche as AWS Cloudwatch Metrics, Cloudwatch Alerts, Stackdriver Metrics, StatsD, Datadog, Nagios, vSphere Metrics and others. It’s open source and has a powerful query language. The front end is the familiar Grafana, and comes with lots of dashboards available out of the box. Prometheus defaults to a pull model, and adding a sidecar exporter is common. Unfortunately the JMX metrics are best captured from inside the JVM, so we can’t just scrape <code>/metrics</code>. Let’s plug in the <a href="https://github.com/prometheus/jmx_exporter">open source jmx_exporter</a> from Prometheus by adding it to the container image.</p>
<h3 id="add-the-prometheus-jmx_exporter-to-the-container">Add the Prometheus jmx_exporter to the container</h3>
<pre><code>-- images/camunda-bpm/Dockerfile     
FROM camunda/camunda-bpm-platform:tomcat-7.11.0

## Add prometheus exporter
RUN wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.11.0/jmx_prometheus_javaagent-0.11.0.jar -P lib/
#9404 is the reserved prometheus-jmx port
ENV CATALINA_OPTS -javaagent:lib/jmx_prometheus_javaagent-0.11.0.jar=9404:/etc/config/prometheus-jmx.yaml
</code></pre>
<p>Well that was easy. The exporter will monitor tomcat and expose it’s metrics at <code>&lt;svc&gt;:9404/metrics</code></p>
<h4 id="configure-the-exporter">Configure the exporter</h4>
<p>The sharp-eyed reader may wonder where that <code>prometheus-jmx.yaml</code> is coming from. There are many different things that can run in a JVM, and tomcat is just one of them, so the exporter needs some configuration. Standard configurations for tomcat, wildfly, kafka, etc are available <a href="https://github.com/prometheus/jmx_exporter/tree/master/example_configs">here</a>. We’ll add the tomcat one as a <a href="https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/">ConfigMap</a> in Kubernetes, and then mount it as a volume.</p>
<p>First, we add the tomcat-flavored exporter config file to our <code>platform/config/</code> directory:</p>
<pre><code>platform/config
└── prometheus-jmx.yaml
</code></pre>
<p>Then we add a <a href="%5Bhttps://github.com/kubernetes-sigs/kustomize/blob/master/examples/configGeneration.md%5D(https://github.com/kubernetes-sigs/kustomize/blob/master/examples/configGeneration.md)">ConfigMapGenerator</a> to <code>kustomization.yaml</code>:</p>
<pre><code>-- platform/kustomization.yaml      
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
[...]
configMapGenerator:
  - name: config
    files:
      - config/prometheus-jmx.yaml
</code></pre>
<p>This will add each element of <code>files[]</code> as an element of the <code>config</code> ConfigMap. ConfigMapGenerators are great because they hash the data in the config and trigger a pod restart if it changes. They also reduce the amount of configuration in the Deployment, as you can mount the whole “folder” of config files in one VolumeMount.</p>
<p>Finally we need to mount the ConfigMap as a volume to the pod:</p>
<pre><code>-- platform/deployment.yaml
apiVersion: apps/v1
kind: Deployment
[...]
spec:
  template:
    spec:
      volumes:
        - name: config
          configMap:
            name: config
            defaultMode: 0744
      containers:
        - name: camunda-bpm
          volumeMounts:
            - mountPath: /etc/config/
              name: config
</code></pre>
<p>Nice. All files we add to the ConfigMapGenerator will be exposed in the new <code>/etc/config</code>T directory.<br>
You can extend this pattern to mount any other configuration files you need.</p>
<h3 id="logs">Logs</h3>
<p>Great news! The application logs are already available, for example with <code>kubectl logs</code>. Fluentd (installed by default on GKE) will forward your logs to Elasticsearch, Loki, or your enterprise log platform. If you want to jsonify your logs, you could follow the pattern above to <a href="https://forum.camunda.org/t/camunda-json-logging-for-shared-engine-solved/8651">set up logback.</a></p>
<h1 id="how-to-run-on-kubernetes---manifests-plain">How to run on Kubernetes - manifests (plain)</h1>
<blockquote>
<p><strong>Note:</strong> Kustomize variable handling is still relatively new. Although kustomize is now included in kubectl as <code>kubectl apply -k</code>, as of 1.14.2 you still need to <code>go get sigs.k8s.io/kustomize</code> for the <code>$HOSTNAME</code> to work. If that’s not an option, you can hardcode the variable references in  <code>ingress-patch.yaml</code></p>
</blockquote>
<blockquote>
<p><strong>Note:</strong> Secret value plugins are <a href="https://github.com/kubernetes-sigs/kustomize/blob/master/examples/secretGeneratorPlugin.md#secret-values-from-anywhere">available now in kustomize</a>. This allows you to populate them from sources such as existing kube-secrets or vault. Check it out!</p>
</blockquote>
<div class="mermaid"><svg xmlns="http://www.w3.org/2000/svg" id="mermaid-svg-EjYRbPa81nMnuIfw" height="100%" width="100%" style="max-width:850px;" viewBox="-50 -10 850 336"><g></g><g><line id="actor108" x1="75" y1="5" x2="75" y2="325" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="0" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="75" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="75" dy="0">skaffold</tspan></text></g><g><line id="actor109" x1="275" y1="5" x2="275" y2="325" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="200" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="275" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="275" dy="0">google cloud build</tspan></text></g><g><line id="actor110" x1="475" y1="5" x2="475" y2="325" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="400" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="475" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="475" dy="0">kustomize</tspan></text></g><g><line id="actor111" x1="675" y1="5" x2="675" y2="325" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="600" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="675" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="675" dy="0">kubernetes</tspan></text></g><defs><marker id="arrowhead" refX="5" refY="2" markerWidth="6" markerHeight="4" orient="auto"><path d="M 0,0 V 4 L6,2 Z"></path></marker></defs><defs><marker id="crosshead" markerWidth="15" markerHeight="8" orient="auto" refX="16" refY="4"><path fill="black" stroke="#000000" stroke-width="1px" d="M 9,2 V 6 L16,4 Z" style="stroke-dasharray: 0, 0;"></path><path fill="none" stroke="#000000" stroke-width="1px" d="M 0,1 L 6,7 M 6,1 L 0,7" style="stroke-dasharray: 0, 0;"></path></marker></defs><g><text x="175" y="93" class="messageText" style="text-anchor: middle;">upload Dockerfile &amp; context</text><line x1="75" y1="100" x2="275" y2="100" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><text x="175" y="128" class="messageText" style="text-anchor: middle;">return built image tag</text><line x1="275" y1="135" x2="75" y2="135" class="messageLine1" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line></g><g><text x="275" y="163" class="messageText" style="text-anchor: middle;">rewrite manifests with new tag</text><line x1="75" y1="170" x2="475" y2="170" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><text x="575" y="198" class="messageText" style="text-anchor: middle;">upload manifests</text><line x1="475" y1="205" x2="675" y2="205" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><text x="475" y="233" class="messageText" style="text-anchor: middle;">pull new image from gcr.io</text><line x1="675" y1="240" x2="275" y2="240" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><rect x="0" y="260" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="75" y="292.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="75" dy="0">skaffold</tspan></text></g><g><rect x="200" y="260" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="275" y="292.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="275" dy="0">google cloud build</tspan></text></g><g><rect x="400" y="260" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="475" y="292.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="475" dy="0">kustomize</tspan></text></g><g><rect x="600" y="260" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="675" y="292.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="675" dy="0">kubernetes</tspan></text></g></svg></div>
<div class="mermaid"><svg xmlns="http://www.w3.org/2000/svg" id="mermaid-svg-6rNkcmZHBFdQ3WFP" height="100%" width="100%" style="max-width:850px;" viewBox="-50 -10 850 336"><g></g><g><line id="actor112" x1="75" y1="5" x2="75" y2="325" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="0" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="75" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="75" dy="0">skaffold</tspan></text></g><g><line id="actor113" x1="275" y1="5" x2="275" y2="325" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="200" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="275" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="275" dy="0">google cloud build</tspan></text></g><g><line id="actor114" x1="475" y1="5" x2="475" y2="325" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="400" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="475" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="475" dy="0">kustomize</tspan></text></g><g><line id="actor115" x1="675" y1="5" x2="675" y2="325" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="600" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="675" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="675" dy="0">kubernetes</tspan></text></g><defs><marker id="arrowhead" refX="5" refY="2" markerWidth="6" markerHeight="4" orient="auto"><path d="M 0,0 V 4 L6,2 Z"></path></marker></defs><defs><marker id="crosshead" markerWidth="15" markerHeight="8" orient="auto" refX="16" refY="4"><path fill="black" stroke="#000000" stroke-width="1px" d="M 9,2 V 6 L16,4 Z" style="stroke-dasharray: 0, 0;"></path><path fill="none" stroke="#000000" stroke-width="1px" d="M 0,1 L 6,7 M 6,1 L 0,7" style="stroke-dasharray: 0, 0;"></path></marker></defs><g><text x="175" y="93" class="messageText" style="text-anchor: middle;">upload Dockerfile &amp; context</text><line x1="75" y1="100" x2="275" y2="100" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><text x="175" y="128" class="messageText" style="text-anchor: middle;">return built image tag</text><line x1="275" y1="135" x2="75" y2="135" class="messageLine1" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line></g><g><text x="275" y="163" class="messageText" style="text-anchor: middle;">rewrite manifests with new tag</text><line x1="75" y1="170" x2="475" y2="170" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><text x="575" y="198" class="messageText" style="text-anchor: middle;">upload manifests</text><line x1="475" y1="205" x2="675" y2="205" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><text x="475" y="233" class="messageText" style="text-anchor: middle;">pull new image from gcr.io</text><line x1="675" y1="240" x2="275" y2="240" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><rect x="0" y="260" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="75" y="292.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="75" dy="0">skaffold</tspan></text></g><g><rect x="200" y="260" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="275" y="292.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="275" dy="0">google cloud build</tspan></text></g><g><rect x="400" y="260" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="475" y="292.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="475" dy="0">kustomize</tspan></text></g><g><rect x="600" y="260" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="675" y="292.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="675" dy="0">kubernetes</tspan></text></g></svg></div>
<pre><code>.
├── Makefile
├── images
│   └── camunda-bpm
│       └── Dockerfile
├── ingress-patch.yaml
├── kustomization.yaml
├── namespace.yaml
├── platform
│   ├── config
│   │   └── prometheus-jmx.yaml
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── kustomization.yaml
│   └── service.yaml
├── site-data.yaml
├── site-data.yaml.example
└── skaffold.yaml
</code></pre>
<p>Install skaffold</p>
<p><a href="https://skaffold.dev/docs/getting-started/#installing-skaffold">https://skaffold.dev/docs/getting-started/#installing-skaffold</a></p>
<p>Set up credentials</p>
<p><code>gcloud auth application-default login</code> since skaffold uses the SDK</p>
<p>Need to scrub camunda-cloud-240911 from files</p>
<p>Use xmlstarlet</p>
<p>Include whole manifest for those who don’t want kustomise</p>
<p>Note about prometheus configuration servicemonitors<br>
json logging</p>
<h2 id="questions">Questions?</h2>
<p>Please ask questions specific to Camunda on our <a href="http://forum.camunda.org">forum</a>!  Questions about Kubernetes may be better asked on the <a href="https://slack.k8s.io/">k8s slack</a>.</p>
<h1 id="welcome-to-stackedit">Welcome to StackEdit!</h1>
<p>Hi! I’m your first Markdown file in <strong>StackEdit</strong>. If you want to learn about StackEdit, you can read me. If you want to play with Markdown, you can edit me. Once you have finished with me, you can create new files by opening the <strong>file explorer</strong> on the left corner of the navigation bar.</p>
<h1 id="files">Files</h1>
<p>StackEdit stores your files in your browser, which means all your files are automatically saved locally and are accessible <strong>offline!</strong></p>
<h2 id="create-files-and-folders">Create files and folders</h2>
<p>The file explorer is accessible using the button in left corner of the navigation bar. You can create a new file by clicking the <strong>New file</strong> button in the file explorer. You can also create folders by clicking the <strong>New folder</strong> button.</p>
<h2 id="switch-to-another-file">Switch to another file</h2>
<p>All your files are listed in the file explorer. You can switch from one to another by clicking a file in the list.</p>
<h2 id="rename-a-file">Rename a file</h2>
<p>You can rename the current file by clicking the file name in the navigation bar or by clicking the <strong>Rename</strong> button in the file explorer.</p>
<h2 id="delete-a-file">Delete a file</h2>
<p>You can delete the current file by clicking the <strong>Remove</strong> button in the file explorer. The file will be moved into the <strong>Trash</strong> folder and automatically deleted after 7 days of inactivity.</p>
<h2 id="export-a-file">Export a file</h2>
<p>You can export the current file by clicking <strong>Export to disk</strong> in the menu. You can choose to export the file as plain Markdown, as HTML using a Handlebars template or as a PDF.</p>
<h1 id="synchronization">Synchronization</h1>
<p>Synchronization is one of the biggest features of StackEdit. It enables you to synchronize any file in your workspace with other files stored in your <strong>Google Drive</strong>, your <strong>Dropbox</strong> and your <strong>GitHub</strong> accounts. This allows you to keep writing on other devices, collaborate with people you share the file with, integrate easily into your workflow… The synchronization mechanism takes place every minute in the background, downloading, merging, and uploading file modifications.</p>
<p>There are two types of synchronization and they can complement each other:</p>
<ul>
<li>
<p>The workspace synchronization will sync all your files, folders and settings automatically. This will allow you to fetch your workspace on any other device.</p>
<blockquote>
<p>To start syncing your workspace, just sign in with Google in the menu.</p>
</blockquote>
</li>
<li>
<p>The file synchronization will keep one file of the workspace synced with one or multiple files in <strong>Google Drive</strong>, <strong>Dropbox</strong> or <strong>GitHub</strong>.</p>
<blockquote>
<p>Before starting to sync files, you must link an account in the <strong>Synchronize</strong> sub-menu.</p>
</blockquote>
</li>
</ul>
<h2 id="open-a-file">Open a file</h2>
<p>You can open a file from <strong>Google Drive</strong>, <strong>Dropbox</strong> or <strong>GitHub</strong> by opening the <strong>Synchronize</strong> sub-menu and clicking <strong>Open from</strong>. Once opened in the workspace, any modification in the file will be automatically synced.</p>
<h2 id="save-a-file">Save a file</h2>
<p>You can save any file of the workspace to <strong>Google Drive</strong>, <strong>Dropbox</strong> or <strong>GitHub</strong> by opening the <strong>Synchronize</strong> sub-menu and clicking <strong>Save on</strong>. Even if a file in the workspace is already synced, you can save it to another location. StackEdit can sync one file with multiple locations and accounts.</p>
<h2 id="synchronize-a-file">Synchronize a file</h2>
<p>Once your file is linked to a synchronized location, StackEdit will periodically synchronize it by downloading/uploading any modification. A merge will be performed if necessary and conflicts will be resolved.</p>
<p>If you just have modified your file and you want to force syncing, click the <strong>Synchronize now</strong> button in the navigation bar.</p>
<blockquote>
<p><strong>Note:</strong> The <strong>Synchronize now</strong> button is disabled if you have no file to synchronize.</p>
</blockquote>
<h2 id="manage-file-synchronization">Manage file synchronization</h2>
<p>Since one file can be synced with multiple locations, you can list and manage synchronized locations by clicking <strong>File synchronization</strong> in the <strong>Synchronize</strong> sub-menu. This allows you to list and remove synchronized locations that are linked to your file.</p>
<h1 id="publication">Publication</h1>
<p>Publishing in StackEdit makes it simple for you to publish online your files. Once you’re happy with a file, you can publish it to different hosting platforms like <strong>Blogger</strong>, <strong>Dropbox</strong>, <strong>Gist</strong>, <strong>GitHub</strong>, <strong>Google Drive</strong>, <strong>WordPress</strong> and <strong>Zendesk</strong>. With <a href="http://handlebarsjs.com/">Handlebars templates</a>, you have full control over what you export.</p>
<blockquote>
<p>Before starting to publish, you must link an account in the <strong>Publish</strong> sub-menu.</p>
</blockquote>
<h2 id="publish-a-file">Publish a File</h2>
<p>You can publish your file by opening the <strong>Publish</strong> sub-menu and by clicking <strong>Publish to</strong>. For some locations, you can choose between the following formats:</p>
<ul>
<li>Markdown: publish the Markdown text on a website that can interpret it (<strong>GitHub</strong> for instance),</li>
<li>HTML: publish the file converted to HTML via a Handlebars template (on a blog for example).</li>
</ul>
<h2 id="update-a-publication">Update a publication</h2>
<p>After publishing, StackEdit keeps your file linked to that publication which makes it easy for you to re-publish it. Once you have modified your file and you want to update your publication, click on the <strong>Publish now</strong> button in the navigation bar.</p>
<blockquote>
<p><strong>Note:</strong> The <strong>Publish now</strong> button is disabled if your file has not been published yet.</p>
</blockquote>
<h2 id="manage-file-publication">Manage file publication</h2>
<p>Since one file can be published to multiple locations, you can list and manage publish locations by clicking <strong>File publication</strong> in the <strong>Publish</strong> sub-menu. This allows you to list and remove publication locations that are linked to your file.</p>
<h1 id="markdown-extensions">Markdown extensions</h1>
<p>StackEdit extends the standard Markdown syntax by adding extra <strong>Markdown extensions</strong>, providing you with some nice features.</p>
<blockquote>
<p><strong>ProTip:</strong> You can disable any <strong>Markdown extension</strong> in the <strong>File properties</strong> dialog.</p>
</blockquote>
<h2 id="smartypants">SmartyPants</h2>
<p>SmartyPants converts ASCII punctuation characters into “smart” typographic punctuation HTML entities. For example:</p>

<table>
<thead>
<tr>
<th></th>
<th>ASCII</th>
<th>HTML</th>
</tr>
</thead>
<tbody>
<tr>
<td>Single backticks</td>
<td><code>'Isn't this fun?'</code></td>
<td>‘Isn’t this fun?’</td>
</tr>
<tr>
<td>Quotes</td>
<td><code>"Isn't this fun?"</code></td>
<td>“Isn’t this fun?”</td>
</tr>
<tr>
<td>Dashes</td>
<td><code>-- is en-dash, --- is em-dash</code></td>
<td>– is en-dash, — is em-dash</td>
</tr>
</tbody>
</table><h2 id="katex">KaTeX</h2>
<p>You can render LaTeX mathematical expressions using <a href="https://khan.github.io/KaTeX/">KaTeX</a>:</p>
<p>The <em>Gamma function</em> satisfying <span class="katex--inline"><span class="katex"><span class="katex-mathml"><math><semantics><mrow><mi mathvariant="normal">Γ</mi><mo>(</mo><mi>n</mi><mo>)</mo><mo>=</mo><mo>(</mo><mi>n</mi><mo>−</mo><mn>1</mn><mo>)</mo><mo>!</mo><mspace width="1em"></mspace><mi mathvariant="normal">∀</mi><mi>n</mi><mo>∈</mo><mi mathvariant="double-struck">N</mi></mrow><annotation encoding="application/x-tex">\Gamma(n) = (n-1)!\quad\forall n\in\mathbb N</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height: 1em; vertical-align: -0.25em;"></span><span class="mord">Γ</span><span class="mopen">(</span><span class="mord mathit">n</span><span class="mclose">)</span><span class="mspace" style="margin-right: 0.277778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right: 0.277778em;"></span></span><span class="base"><span class="strut" style="height: 1em; vertical-align: -0.25em;"></span><span class="mopen">(</span><span class="mord mathit">n</span><span class="mspace" style="margin-right: 0.222222em;"></span><span class="mbin">−</span><span class="mspace" style="margin-right: 0.222222em;"></span></span><span class="base"><span class="strut" style="height: 1em; vertical-align: -0.25em;"></span><span class="mord">1</span><span class="mclose">)</span><span class="mclose">!</span><span class="mspace" style="margin-right: 1em;"></span><span class="mord">∀</span><span class="mord mathit">n</span><span class="mspace" style="margin-right: 0.277778em;"></span><span class="mrel">∈</span><span class="mspace" style="margin-right: 0.277778em;"></span></span><span class="base"><span class="strut" style="height: 0.68889em; vertical-align: 0em;"></span><span class="mord mathbb">N</span></span></span></span></span> is via the Euler integral</p>
<p><span class="katex--display"><span class="katex-display"><span class="katex"><span class="katex-mathml"><math><semantics><mrow><mi mathvariant="normal">Γ</mi><mo>(</mo><mi>z</mi><mo>)</mo><mo>=</mo><msubsup><mo>∫</mo><mn>0</mn><mi mathvariant="normal">∞</mi></msubsup><msup><mi>t</mi><mrow><mi>z</mi><mo>−</mo><mn>1</mn></mrow></msup><msup><mi>e</mi><mrow><mo>−</mo><mi>t</mi></mrow></msup><mi>d</mi><mi>t</mi>&amp;ThinSpace;<mi mathvariant="normal">.</mi></mrow><annotation encoding="application/x-tex">
\Gamma(z) = \int_0^\infty t^{z-1}e^{-t}dt\,.
</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height: 1em; vertical-align: -0.25em;"></span><span class="mord">Γ</span><span class="mopen">(</span><span class="mord mathit" style="margin-right: 0.04398em;">z</span><span class="mclose">)</span><span class="mspace" style="margin-right: 0.277778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right: 0.277778em;"></span></span><span class="base"><span class="strut" style="height: 2.32624em; vertical-align: -0.91195em;"></span><span class="mop"><span class="mop op-symbol large-op" style="margin-right: 0.44445em; position: relative; top: -0.001125em;">∫</span><span class="msupsub"><span class="vlist-t vlist-t2"><span class="vlist-r"><span class="vlist" style="height: 1.41429em;"><span class="" style="top: -1.78805em; margin-left: -0.44445em; margin-right: 0.05em;"><span class="pstrut" style="height: 2.7em;"></span><span class="sizing reset-size6 size3 mtight"><span class="mord mtight">0</span></span></span><span class="" style="top: -3.8129em; margin-right: 0.05em;"><span class="pstrut" style="height: 2.7em;"></span><span class="sizing reset-size6 size3 mtight"><span class="mord mtight">∞</span></span></span></span><span class="vlist-s">​</span></span><span class="vlist-r"><span class="vlist" style="height: 0.91195em;"><span class=""></span></span></span></span></span></span><span class="mspace" style="margin-right: 0.166667em;"></span><span class="mord"><span class="mord mathit">t</span><span class="msupsub"><span class="vlist-t"><span class="vlist-r"><span class="vlist" style="height: 0.864108em;"><span class="" style="top: -3.113em; margin-right: 0.05em;"><span class="pstrut" style="height: 2.7em;"></span><span class="sizing reset-size6 size3 mtight"><span class="mord mtight"><span class="mord mathit mtight" style="margin-right: 0.04398em;">z</span><span class="mbin mtight">−</span><span class="mord mtight">1</span></span></span></span></span></span></span></span></span><span class="mord"><span class="mord mathit">e</span><span class="msupsub"><span class="vlist-t"><span class="vlist-r"><span class="vlist" style="height: 0.843556em;"><span class="" style="top: -3.113em; margin-right: 0.05em;"><span class="pstrut" style="height: 2.7em;"></span><span class="sizing reset-size6 size3 mtight"><span class="mord mtight"><span class="mord mtight">−</span><span class="mord mathit mtight">t</span></span></span></span></span></span></span></span></span><span class="mord mathit">d</span><span class="mord mathit">t</span><span class="mspace" style="margin-right: 0.166667em;"></span><span class="mord">.</span></span></span></span></span></span></p>
<blockquote>
<p>You can find more information about <strong>LaTeX</strong> mathematical expressions <a href="http://meta.math.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference">here</a>.</p>
</blockquote>
<h2 id="uml-diagrams">UML diagrams</h2>
<p>You can render UML diagrams using <a href="https://mermaidjs.github.io/">Mermaid</a>. For example, this will produce a sequence diagram:</p>

