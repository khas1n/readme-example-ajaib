# React Module Boilerplate

Repository ini berisi Boilerplate untuk Module di aplikasi `microfrontend-poc` 
dengan framework `React` dan `Typescript`, Hasil dari module ini bertipe `remote` dan nantinya akan berjalan dan menjadi bagian dari `Host`[(Container Module)](https://gitlab.com/dpr-poc/front-end/container)

Aplikasi ini menggunakan konsep [Module Federation](https://webpack.js.org/concepts/module-federation/) dari Webpack 5+ 
## Tech Stack

**Fornt-end Framework:** [React](https://reactjs.org/)

**CSS Framework:** [Tailwind](https://tailwindcss.com/)

**Bundler:** [Webpack 5](https://webpack.js.org/)

**Package Manager:** [PNPM](https://pnpm.io/)

## Prerequisite

Pastikan di mesin anda sudah terinstall
* Node 16.13.1 or higher
* PNPM

## How To Use
Berikut bagaimana cara memakai boilerplate ini : 

### 1. Konfigurasi Port di development/isolation mode
Anda harus memastikan port yang akan dipakai unik dan tidak pernah terpakai 
di module-module lainnya di Aplikasi `microfrontend-poc` dan mengatur nya di 
`config/webpack.dev.js` pada key `devServer -> port`

```javascript
....
devServer: {
    port: PORT_UNIK_ANDA,
    ....
},
.....
```
dan pastikan untuk port pada key `publicPath` sama dengan `PORT_UNIK_ANDA` karena ini 
akan menjadi cara bagaimana `Host` mengakses module ini pada saat development/isolation mode
```javascript
....
output: {
    publicPath: `http://localhost:PORT_UNIK_ANDA/`,
},
....
```

### 2. Konfigurasi Bootstrap module
Setiap aplikasi react harus memiliki root element yang digunakan untuk 
me-render react aplikasi tersebut. Maka kita akan memilih nama id untuk root element yang sama dengan nama module ini

contoh nama yang di pilih adalah `example` maka kita akan memberi id pada root element kita `_example_root` dan akan 
dimasukan kedalam file `public/index.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    ....
    <title>Example isolation</title>
    ....
  </head>
   <body>
    <div id="_example-root"></div>
  </body>
</html>
```

setelah root element diatur maka kita akan mengatur `src/bootstrap.js` berdasarkan nama id element yang di pilih
untuk development/isolation mode
```javascript
// If we are in development and in isolation,
// call mount immediately
if (process.env.NODE_ENV === "development") {
  const devRoot = document.querySelector("#_example-root");

  if (devRoot) {
    mount(devRoot, {
      defaultHistory: createBrowserHistory(),
    });
  }
}
```

### 3. Konfigurasi Tailwind CSS Prefix
Untuk menjaga agar tidak tumpang tindih css, maka kita akan menggunakan prefix dan fitur `important` untuk spesifik module

contoh kita menggunakan nama `example` dan id root element `#_example-root`, maka kita akan mengaturnya di file `tailwind.config.js`

```javascript
module.exports = {
    ....
    important: "#_example-root",
    prefix: "ep-",
}
```

maka untuk cara menggunakan utility css tailwind dengan sebagai berikut:

```
<div className="ep-flex ep-w-96 ep-max-w-md ep-flex-col">
```

### 4. Konfigurasi Module Federation
setelah itu atur `Module Federation` pada key `plugins`, pastikan memilih nama yang unik karena
ini akan menentukan bagaimana cara `module` ini akan dipanggil di `Host`.

contoh nama yang anda pilih adalah `example` maka anda juga harus memberi `exampleApp` di key `exposes`

```javascript
plugins: [
    new ModuleFederationPlugin({
      name: "example",
      filename: "remoteEntry.js",
      exposes: {
        "./ExampleApp": "./src/bootstrap",
      },
      shared: packageJson.dependencies,
    }),
....
]
```

### 5. Konfigurasi Production
Konfigurasi Production bisa dilihat pada file `config/webpack.prod.js`

contoh kita akan menggunakan nama `example`

```javascript
const prodConfig = {
  mode: "production",
  output: {
    filename: "[name].[contenthash].js",
    publicPath: "/example/latest/",
    clean: true,
  },
  plugins: [
    new ModuleFederationPlugin({
      name: "example",
      filename: "remoteEntry.js",
      exposes: {
        "./ExampleApp": "./src/bootstrap",
      },
      shared: packageJson.dependencies,
    }),
  ],
};
```

### 5. Konfigurasi di Host
Untuk mendaftarkan module yang kita buat ke `Host` maka kita akan mengubah beberapa file yang berada di `Host`[(Container Module)](https://gitlab.com/dpr-poc/front-end/container)
* container/config/webpack.dev.js: Mendaftarkan module kita di development/isolation mode 
* container/config/webpack.prod.js: Mendaftarkan module kita untuk production
* menambahkan file untuk memuat module kita di dalam folder `container/src/components/ModuleFederation`
* container/src/App.tsx : Mendaftarkan module kita pada `route` yang tersedia

contoh kita akan menambahkan Module Example
* container/config/webpack.dev.js:
```javascript
const devConfig = {
    ....
    plugins: [
        ....
        new ModuleFederationPlugin({
            name: "container",
            remotes: {
                ....
                example: "example@http://localhost:PORT_UNIK_ANDA/remoteEntry.js",
                ....
            },
        }),
        ....
    ]
    ....
}
```

* container/config/webpack.prod.js:
```javascript
const devConfig = {
    ....
    plugins: [
        ....
        new ModuleFederationPlugin({
            name: "container",
            remotes: {
                ....
                example: `example@${domain}/example/remoteEntry.js`,
                ....
            },
        }),
        ....
    ]
    ....
}
```

* menambahkan file untuk memuat module kita di dalam folder `container/src/components/ModuleFederation` dengan nama file `ExampleApp.tsx`
```javascript
import React, { useRef, useEffect } from "react";
import { mount } from "example/ExampleApp"; // berdasarkan apa yang kita daftar di file konfigurasi
import { useLocation } from "react-router-dom";
import { History } from "history";

import history from "../../history";

const ExampleApp: React.FC = () => {
  const ref = useRef(null);
  const location = useLocation();

  useEffect(() => {
    const { onParentNavigate } = mount(ref.current, {
      initialPath: location.pathname,
      onNavigate: ({ location: { pathname: nextPathName } }: History) => {
        const { pathname } = location;
        if (pathname !== nextPathName) {
          history.push(nextPathName);
        }
      },
    });
    const unlisten = history.listen(() => onParentNavigate);
    return () => {
      unlisten();
    };
  }, []);
  return <div id="_example-root" ref={ref} />;
};

export default ExampleApp;

```

* container/src/App.tsx
Kita akan memanfaatkan fitur LazyLoad pada react dan daftarkan module example kita ke dalam route, dan module kita akan bertanggung jawab untuk route `/example/**`
```jsx
const ExampleLazy = lazy(() => import("./components/ModuleFederation/ExampleApp"));
export default () => {
    return (
        ....
        <HistoryRouter history={history}>
            <Routes>
                ....
                <Route path="/" element={<RequireAuth><Layout /></RequireAuth>}>
                    ....
                    <Route path="/example/*" element={
                            <Suspense fallback={<LoadingIndicatorModule />}>
                                <ExampleLazy />
                            </Suspense>
                        }
                    />
                    ....
                </Route>
                ....
            </Routes>
        </HistoryRouter>
        ....
    )
}
```