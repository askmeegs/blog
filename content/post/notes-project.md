---
title: "Building My Own Notes App (Glitch, Kubernetes)"
date: "2019-05-19"
---

*TLDR: Code is [here](https://glitch.com/edit/#!/meegs) (frontend) and [here](https://github.com/askmeegs/notes) (backend).*

I've used lots of notes apps. [Simplenote](https://simplenote.com/). [nvALT](https://brettterpstra.com/projects/nvalt/). [Bear](https://bear.app/). Apple Notes. And they've worked well enough. But lately I've been looking for two things:

1) A programmatic way to read/write notes (API)
2) A way to see a random note from the past (for inspiration)

Right now, I'm using Apple Notes across all my devices, which is locked down re: programmatic access. So I knew it was time for a change. To take a first stab at the problem, I wrote my own private notes server, with an accompanying frontend.

## Design

My requirements were minimal:
1) Create + view notes. (And notes can be immutable; no need to edit.)
1) View a random note
1) HTTPS everywhere. Also, the frontend must be locked down with a login screen.

Here is where I landed:

![architecture](/media/architecture.png)

## 💻 Frontend

I chose to use [Glitch](https://glitch.com/about/) to create (and host) my notes frontend. I've heard great things about Glitch — it's a community of developers/apps, but also way to create free, fully-hosted web apps without having to deal with `npm`, etc. on my own machine. I think my use case (a "hacking on the web" / side-project sort of app) was a great fit for Glitch, and I'd definitely recommend it.

My JavaScript is rusty, so writing the frontend took the most time out of this whole process. ([Code here](https://glitch.com/edit/#!/meegs).) For the server-side, I settled on a set of [express.js](https://expressjs.com/) endpoints (node.js), rendered via [pug.js](https://pugjs.org/api/getting-started.html). The express functions use [axios.js](https://flaviocopes.com/axios/) to call the backend notes server.

On the client-side, I used [jQuery](https://jquery.com/) for my button listeners. I also used [DataTables](https://datatables.net/) to add search (and sorting) to my list of notes:

<img src="/media/create.gif">

My favorite thing about writing the frontend was creating the Pug views. So simple! Here's where I show a random note:

```
    div.one
      if error
        h4= error
      if note
        h1 a note from the past
        div.one
          p.randomText= note.note
```

... Which renders to:

<img src="/media/random.gif">

Finally, the whole frontend is locked down with a login screen, which authenticates with the backend, and stores a [JWT](https://jwt.io/) as a cookie (I know, not the most secure). This cookie is used for all subsequent calls to the Backend...

## ☸️ Backend

I *could* have just had my Node.js backend call the database directly. But for more practice, I wrote a server backend in Golang ([code here](https://github.com/askmeegs/notes)). Then, I deployed the backend to Kubernetes, beneath a domain name I had lying around.

The backend is a dead-simple golang HTTP server that speaks JSON (I didn't use any API frameworks or generators). For instance, the handler for getting all the notes looks like this:

```
func GetNotesHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Access-Control-Allow-Origin", "*")
	w.Header().Set("Content-Type", "application/json")

	if err := validateJwt(r); err != nil {
		w.WriteHeader(http.StatusUnauthorized)
		io.WriteString(w, err.Error())
		return
	}

	notes, err := getNotesHelper()
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		io.WriteString(w, err.Error())
	}

	output, _ := json.Marshal(notes)

	w.WriteHeader(http.StatusOK)
	io.WriteString(w, string(output))
}
```

All calls to the server must have a valid JWT, issued via the `POST /login` endpoint.

Then, I wrote a simple [Dockerfile](https://github.com/askmeegs/notes/blob/master/Dockerfile) to containerize the server, then pushed it to my image repo (in Google Container Registry).

Next, I knew I'd have to deploy this Docker container somewhere. I know [Kubernetes](https://kubernetes.io/), so I chose to deploy to [GKE](https://cloud.google.com/kubernetes-engine/) out of convenience. But Kubernetes was probably overkill for my use case (see: [relevant tweet](https://twitter.com/dexhorthy/status/856639005462417409?lang=en)), because I'm the only user of my backend server, and I probably won't generate a lot of requests.

Using GKE, however, allowed me to set up [Managed SSL certificates](https://cloud.google.com/kubernetes-engine/docs/how-to/managed-certs#creating_an_ingress_with_a_managed_certificate) for my domain, using the Kubernetes Ingress resource, and a GCP static IP address. Meaning I could hit `https://<my-domain-name>` and reach my backend server. This came in handy when it came time to connect my Glitch frontend to the backend (Glitch doesn't allow plain `http` calls.). The alternative, here, would be to use [LetsEncrypt](https://letsencrypt.org/) or another certificate authority for your domain.

Finally, here's my Kubernetes [deployment YAML](https://github.com/askmeegs/notes/blob/master/kubernetes/deployment.yaml#L21), which actually deploys the backend server container into my Kubernetes cluster:

```
     containers:
      - name: server
        image:  gcr.io/notesdb/notes:v0.0.4
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: firestore-key
          mountPath: /var/secrets/google
```

What's that `/var/secrets/google`? It's a mounted-in Kubernetes [secret](https://kubernetes.io/docs/concepts/configuration/secret/), which I added to my cluster ahead of time. This secret contains the credentials needed to talk to the database...

## 📝 Database

My notes have to live somewhere, so for a database I chose [Firestore](https://cloud.google.com/firestore/), which is Google Cloud's newer NoSQL database for web and mobile. I chose Firestore mostly because I'd never tried it before; I could have also used Google Cloud [Datastore](https://cloud.google.com/datastore/docs/firestore-or-datastore#in_datastore_mode) or some other NoSQL database.

The DB itself lives in the same Google Cloud project as my Kubernetes cluster, and it's a single collection with a bunch of documents. Each document represents one Note.

<img src="/media/firestore.png">


My backend code uses the Firestore [client library](https://github.com/GoogleCloudPlatform/golang-samples/tree/2c83bbf7c4360188b57fb430e2536eb8bb70c843/firestore) for Golang. This was really easy to use— you just need a [service account](https://firebase.google.com/docs/firestore/quickstart) key, and you can start writing documents to your collection, like this:

```
	n := Note{
		Timestamp: time.Now().Format(time.RFC3339),
		Note:      note.Note,
	}

	_, _, err = client.Collection("notes").Add(context.Background(), n)
```


## That's a wrap!

This was a fun weekend project. And while there's a lot I want to improve (a better mobile UX, render embedded links in the frontend), I'm happy with my app so far.

✨ Thanks for reading!
