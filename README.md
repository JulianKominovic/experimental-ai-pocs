# Experimental AI POCs
## Automatic pull request review

**1. Get the diff from a PR. For example: https://patch-diff.githubusercontent.com/raw/midudev/midu.dev/pull/29.diff**

**2. This returns several diff notations that OpenAI understands.**

**3. Ask `Haz una lista de markdown explicando los cambios más relevantes en unas pocas palabras por línea y otra lista señalando los errores de sintaxis, malas practicas usadas, mejoras posibles, y errores de ortografía:\n`**

**4. Paste what you obtained in step 1.**

## Automatic documentation

**1. Copy code**

**2. Paste it. For example:**

Documenta este custom hook de React:
```tsx
import { useEffect, useRef } from "react";
import { useState } from "react";

type StateMachine<Type> = {
  isLoading: boolean;
  idle: boolean;
  response: {
    data: Type;
  } | null;
  error: {
    message: string;
    statusCode: number;
    error: string;
  } | null;
};

type Return<Type> = StateMachine<Type> & {
  refetch: (cache: boolean) => void;
  abort: () => void;
};

const MAX_RETRIES = 3;
const CACHE_TTL = 600000; //in ms
const RETRY_INTERVAL = 2000;

const useQuery = <Type>(
  url: string,
  requestInfo?: RequestInit
): Return<Type> => {
  const [state, setState] = useState<StateMachine<Type>>({
    isLoading: true,
    idle: true,
    response: null,
    error: null,
  });
  const queryUrl = url;
  const retriesNumber = useRef(0);
  const setResponseData = (res: Type) =>
    setState((prev) => ({
      ...prev,
      idle: false,
      isLoading: false,
      response: { data: res },
      error: null,
    }));
  const setErrorData = (error: Return<Type>["error"]) =>
    setState((prev) => ({
      ...prev,
      idle: false,
      isLoading: false,
      response: null,
      error: error,
    }));
  const abortController = useRef(new AbortController());
  const withCache = useRef(true);

  const abort = () => abortController.current.abort();
  const refetch: Return<Type>["refetch"] = (cache = true) => {
    withCache.current = cache;
    setState((prev) => ({ ...prev, idle: true }));
  };

  useEffect(() => {
    if (!state.isLoading) refetch(true);
  }, [url]);

  const doFetch = async () => {
    const cacheRead = await caches.open("useQuery");
    const found = await cacheRead.match(url);
    const dateHeader: string | null | undefined =
      found?.headers.get("query-date");
    const parseDateHeader: Date = new Date(dateHeader || 0);
    if (
      found &&
      dateHeader &&
      Number(Date.now()) - Number(parseDateHeader) < CACHE_TTL &&
      withCache.current
    ) {
      setResponseData(await found.json());
    } else {
      abortController.current = new AbortController();
      await fetch(queryUrl, {
        ...requestInfo,
        signal: abortController.current.signal,
      })
        .then(async (res) => {
          if (res.ok) {
            const clone = res.clone();
            const cache = await caches.open("useQuery");
            const headers = new Headers(clone.headers);
            headers.append("query-date", new Date().toUTCString());

            await cache.put(
              url,
              new Response(clone.body, { ...clone, headers })
            );
          }

          return await res.json();
        })
        .then((res) => {
          if (res.error) {
            return Promise.reject(res);
          } else {
            setResponseData(res);
          }
          retriesNumber.current = 0;
        })
        .catch(async (err) => {
          if (abortController.current.signal.aborted)
            return setErrorData({
              error: "Aborted by the user",
              message: "Aborted by the user",
              statusCode: -1,
            });

          if (retriesNumber.current < MAX_RETRIES - 1) {
            retriesNumber.current++;
            await new Promise(async (res) => {
              const id = setTimeout(async () => {
                const response = await doFetch();
                clearTimeout(id);
                res(response);
              }, RETRY_INTERVAL);
            });
          } else {
            setErrorData(err);
          }
        })
        .finally(() => {
          withCache.current = true;
        });
    }
  };

  if (state.idle) {
    setState((prev) => ({ ...prev, isLoading: true, idle: false }));
    doFetch();
  }
  return { ...state, refetch, abort };
};

export default useQuery;
```

3. **OpenAI explained how it works **

```
Este custom hook de React se llama useQuery, que acepta una URL y un objeto RequestInit (opcional) y devuelve un objeto con el estado actual de la solicitud, así como funciones para refrescar y abortar la solicitud.

El estado actual de la solicitud se define mediante el tipo de datos genérico Type y se representa mediante el tipo de datos StateMachine<Type>. Este objeto de estado contiene cuatro propiedades:

    isLoading: un booleano que indica si la solicitud está en proceso o no
    idle: un booleano que indica si la solicitud ha sido inicializada o no
    response: un objeto que contiene la respuesta de la solicitud si se ha completado correctamente
    error: un objeto que contiene información de error si se produce un error al realizar la solicitud

Además de estas propiedades, el objeto devuelto por useQuery también tiene dos funciones adicionales:

    refetch: una función que permite volver a enviar la solicitud. Tiene un parámetro opcional cache que indica si se debe usar la caché para la solicitud o no.
    abort: una función que permite cancelar la solicitud en curso.

El código usa useState, useEffect y useRef hooks de React. Además, utiliza la API fetch y la API Cache para leer y escribir datos en caché. También utiliza la API AbortController para permitir la cancelación de la solicitud en curso.

Finalmente, el código tiene algunas variables constantes que controlan el número máximo de intentos (MAX_RETRIES), la duración de la caché (CACHE_TTL) y el intervalo de tiempo entre intentos (RETRY_INTERVAL).
```

but...

**4. Ask `how can I use it in my React component?`**

Para usar el custom hook useQuery en un componente de React, primero debes importarlo en el archivo donde se encuentra el componente. Puedes hacerlo de la siguiente manera:

```tsx

import useQuery from './useQuery'; // ajusta la ruta según la ubicación del archivo useQuery

function MyComponent() {
  const { isLoading, response, error, refetch, abort } = useQuery<MyResponseType>('https://mi-api.com/mi-ruta');

  // ...
}
```
En el ejemplo anterior, se importa el custom hook useQuery y se utiliza en el componente MyComponent. La función useQuery recibe dos argumentos:

    url: una cadena que representa la URL del endpoint de la API que deseas consultar.
    requestInfo: un objeto opcional que representa la configuración de la solicitud que se enviará al endpoint. Este objeto es similar al objeto que se usa con fetch().

El hook retorna un objeto con las siguientes propiedades:

    isLoading: una bandera booleana que indica si la solicitud está en proceso.
    response: un objeto que contiene la respuesta de la API si la solicitud se realizó correctamente.
    error: un objeto que contiene información sobre el error que se produjo durante la solicitud.
    refetch: una función que puedes llamar para volver a realizar la solicitud a la API. Esta función toma un argumento booleano opcional que indica si la solicitud debe leer de la caché o no.
    abort: una función que puedes llamar para abortar la solicitud en curso.

En el ejemplo anterior, la respuesta de la API se define como MyResponseType, que es un tipo genérico que debes reemplazar por el tipo real que esperas recibir de la API.

