---
layout: ~/layouts/MainLayout.astro
title: Points de terminaison
description: Apprenez comment créer des points de terminaison pour servir n'importe quel type de données
i18nReady: true
---
Astro vous laisse la possibilité de créer des points de terminaison ('endpoints' en anglais) personnalisés pour servir n'importe quel type de données. Vou pouvez les utiliser pour générer des images, exposer un document RSS, ou en tant que routes d'API afin de construire une API complète pour votre site.

Dans les sites générés statiquement, vos points de terminaison personnalisés sont appelés lors de la compilation afin de produire des fichiers statiques. Si vous optez pour le mode [SSR](/fr/guides/server-side-rendering/), points de terminaison personnalisés deviendront des points de terminaison "live server" qui seront appelés à la demande. Les points de terminaison Statiques et SSR sont définis de manière similaire, mais les points de terminaison SSR supportent des fonctionnalités additionnelles.

## Points de terminaison statiques

Pour créer un point de terminaison, ajoutez un fichier `.js` ou `.ts` dans le répertoire `/pages`. L'extension `.js` or `.ts` sera supprimée lors du processus de compilation, pour que le nom de fichier inclus l'extension de données que vous souhaitez créer. Par exemple, `src/pages/data.json.ts` générera un point de terminaison `/data.json`.

Les points de terminaison exportent une fonction `get` (optionellement `async`) qui reçoit un [objet context](/fr/reference/api-reference/#endpoint-context) avec des propriétés similaires à l'objet `Astro` global. Elle retourne un objet avec un `body`, et Astro l'appellera au moment de la compilation et utilisera les contenus du body pour générer le fichier.

```ts
// Exemple: src/pages/builtwith.json.ts
// Génère: /builtwith.json
export async function get({params, request}) {
  return {
    body: JSON.stringify({
      name: 'Astro',
      url: 'https://astro.build/',
    }),
  };
}
```

L'objet retourné peut aussi avoir une propriété `encoding`. Cela peut être n'importe quel [`BufferEncoding`](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/bdd02508ddb5eebcf701fdb8ffd6e84eabf47885/types/node/buffer.d.ts#L169) valide accepté par la method Node.js' `fs.writeFile`. Par exemple, pour produire une image binaire png :

```ts title="src/pages/astro-logo.png.ts" {6}
export async function get({ params, request }) {
  const response = await fetch("https://astro.build/assets/press/full-logo-light.png");
  const buffer = Buffer.from(await response.arrayBuffer());
  return {
    body: buffer,
    encoding: 'binary',
  };
}
```

Vous pouvez également saisir vos fonctions de points de terminaison en utilisant le type `APIRoute` :

```ts
import type { APIRoute } from 'astro';

export const get: APIRoute = async function get ({params, request}) {
...
```

### `params` et routage dynamique

Les points de terminaison supportent les même fonctionnalités de [routes dynamiques](/fr/core-concepts/routing/#routes-dynamiques) que les pages. Nommez votre fichier avec un nom de paramètre entre crochets et exportez une [fonction `getStaticPaths()`](/fr/reference/api-reference/#getstaticpaths). Ensuite, vous pouvez accéder au paramètre en utilisant la propriété `params` passé à la fonction du point de terminaison :

```ts title="src/pages/[id].json.ts"
import type { APIRoute } from 'astro';

const usernames = ["Sarah", "Chris", "Dan"]

export const get: APIRoute = ({ params, request }) => {
  const id = params.id;
  return {
    body: JSON.stringify({
      name: usernames[id]
    })
  }
};

export function getStaticPaths () {
  return [ 
    { params: { id: "0"} },
    { params: { id: "1"} },
    { params: { id: "2"} },
  ]
}
```

Ceci générera trois points de terminaison JSON au moment de la compilation : `/api/1.json`, `/api/2.json`, `/api/3.json`. Le routage dynamique avec les points de terminaison fonctionne de la même façon qu'avec les pages, mais comme le point de terminaison est une fonction et pas un composant, les [props](/fr/reference/api-reference/#data-passing-with-props) ne sont pas supportées.

### `request`

Tous les points de terminaison reçoivent une propriété `request`, mais en mode statique, vous avez uniquement accès à `request.url`. Ceci retourne l'URL complète du point de terminaison en cours et fonctionne de la même façon que [Astro.request.url](/fr/reference/api-reference/#astrorequest) le fait pour les pages.

```ts title="src/pages/request-path.json.ts"
import type { APIRoute } from 'astro';

export const get: APIRoute = ({ params, request }) => {
  return {
    body: JSON.stringify({
      path: new URL(request.url).pathname
    })
  };
}
```

## Points de terminaison Serveur(Routes d'API)

Tout ce qui est décrit dans la section des points de terminaison statiques peut également être utilisé en mode SSR : Les fichiers peuvent exporter une fonction `get` qui reçoit un [objet context](/fr/reference/api-reference/#endpoint-context) avec des propriétés similaires à l'objet `Astro` global.

Mais, à la différence du mode `static`, lorsque vous configurez le mode `server`, les points de terminaison seront construits à la demande. Ceci déverrouille de nouvelles fonctionnalités qui ne sont pas disponibles au moment de la compilation, et vous permet de construire des routes d'API qui écoutent les demandes et exécutent le code de manière sécurisée sur le serveur au moment de l'exécution.

:::note
Soyez sûr d'avoir [activé le SSR](/fr/guides/server-side-rendering/#activation-du-mode-ssr-dans-votre-projet) avant d'essayer ces exemples.
:::

Les points de terminaison serveur peuvent accéder aux `params` sans avoir à exporter `getStaticPaths`, et ils peuvent retourner un objet [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response), permettant d'affecter des codes de statut et des en-têtes :

```js title="src/pages/[id].json.js"
import { getProduct } from '../db';

export async function get({ params }) {
  const id = params.id;
  const product = await getProduct(id);

  if (!product) {
    return new Response(null, {
      status: 404,
      statusText: 'Not found'
    });
  }

  return new Response(JSON.stringify(product), {
    status: 200,
    headers: {
      "Content-Type": "application/json"
    }
  });
}
```

Ceci répondra à n'importe quelle requête qui correspond à la route dynamique. Par exemple, si nous naviguons vers `/helmet.json`, `params.id` sera affecté à `helmet`. Si `helmet` existe dans la base de données product d'exemple, le point de terminaison utilisera un objet `Response` pour renvoyer du JSON et retourner un [code de statut HTTP 200](https://developer.mozilla.org/en-US/docs/Web/API/Response/status). Sinon, il utilisera un objet `Response` pour renvoyer une `404`.

### Méthodes HTTP

En plus de la fonction `get`, vous pouvez exporter une fonction avec le nom de n'importe quelle [méthode HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods). Lorsqu'une requête arrive, Astro vérifiera la méthode et appellera la fonction correspondante.

Vous pouvez aussi exporter une fonction `all` pour faire correspondre n'importe quelle méthode qui n'a pas de fonction exportée correspondante. S'il y a une demande avec une méthode qui ne correspond pas, elle sera redirigée vers la [page 404](/fr/core-concepts/astro-pages/#page-derreur-404-personnalisée).

:::note
Etant donné que `delete` est un mot réservé en JavaScript, exportez une fonction `del` pour faire correspondre la méthode delete.
:::

```ts title="src/pages/methods.json.ts"
export const get: APIRoute = ({ params, request }) => {
  return {
    body: JSON.stringify({
      message: "Ceci est un GET !"
    })
  }
};

export const post: APIRoute = ({ request }) => {
  return {
    body: JSON.stringify({
      message: "Ceci est un POST !"
    })
  }
}

export const del: APIRoute = ({ request }) => {
  return {
    body: JSON.stringify({
      message: "Ceci est un DELETE !"
    })
  }
}

export const all: APIRoute = ({ request }) => {
  return {
    body: JSON.stringify({
      message: `Ceci est un ${request.method} !`
    })
  }
}
```

### `request`

En mode SSR, la propriété `request` retourne un objet [`Request`](https://developer.mozilla.org/en-US/docs/Web/API/Request) compètement utilisable qui fait référence à la requête en cours. Cela vous permettra d'accepter les données et vérifier les en-têtes :

```ts title="src/pages/test-post.json.ts"
export const post: APIRoute = async ({ request }) => {
  if (request.headers.get("Content-Type") === "application/json") {
    const body = await request.json();
    const name = body.name;
    return new Response(JSON.stringify({
      message: "Your name was: " + name
    }), {
      status: 200
    })
  }
  return new Response(null, { status: 400 });
}
```

### Redirections

Le contexte du point de terminaison exporte une méthode `redirect()` utilitaire similaire à `Astro.redirect`:

```js title="src/pages/links/[id].js" {14}
import { getLinkUrl } from '../db';

export async function get({ params, redirect }) {
  const { id } = params;
  const link = await getLinkUrl(id);

  if (!link) {
    return new Response(null, {
      status: 404,
      statusText: 'Not found'
    });
  }

  return redirect(link, 307);
}
```

### Exemple: Vérifier un captcha

Les points de terminaison serveur peuvent être utilisés en tant que points de terminaison d'API REST pour exécuter des fonctions telles que des authentifications, de l'accès à une base de données, et des vérifications sans exposer de données sensibles au client.

Dans cet exemple ci-dessous, une route d'API est utilisée pour vérifier un reCAPTCHA Google v3 sans exposer le secret aux clients.

Sur le serveur, nous définissons une méthode post qui accepte les données du recaptcha, puis le vérifie avec l'API reCAPTCHA. Ici, nous pouvons définir les valeurs secrètes ou lire les variables d'environnement de manière sécurisée.

```js title="src/pages/recaptcha.js"
export async function post({ request }) {
  const data = await request.json();

  const recaptchaURL = 'https://www.google.com/recaptcha/api/siteverify';
  const requestBody = {
    secret: "YOUR_SITE_SECRET_KEY",   // Ceci peut être une variable d'environnement
    response: data.recaptcha          // Le token passé par le client
  };

  const response = await fetch(recaptchaURL, {
    method: "POST",
    body: JSON.stringify(requestBody)
  });

  const responseData = await response.json();

  return new Response(JSON.stringify(responseData), { status: 200 });
}
```

Ensuite, nous accédons à notre point de terminaison en utilisant `fetch` depuis un script client :

```astro title="src/pages/index.astro"
<html>
  <head>
    <script src="https://www.google.com/recaptcha/api.js"></script>
  </head>

  <body>
    <button class="g-recaptcha" 
      data-sitekey="PUBLIC_SITE_KEY" 
      data-callback="onSubmit" 
      data-action="submit"> Cliquez pour vérifier le captcha ! </button>

    <script is:inline>
      function onSubmit(token) {
        fetch("/recaptcha", {
          method: "POST",
          body: JSON.stringify({ recaptcha: token })
        })
        .then((response) => response.json())
        .then((gResponse) => {
          if (gResponse.success) {
            // La vérification du Captcha est un succès
          } else {
            // La vérification du Captcha a échoué
          }
        })
      }
    </script>
  </body>
</html>
```
