[![Circle CI](https://circleci.com/gh/kamilchm/go2nix.svg?style=shield)](https://circleci.com/gh/kamilchm/go2nix)

# go2nix - nix packages for Go applications

go2nix is best suited for Go apps written before [dep](https://github.com/golang/dep) or [modules](https://github.com/golang/go/wiki/Modules) were introduced.

- If you see a `Gopkgs.lock` file in the source try [dep2nix](https://github.com/nixcloud/dep2nix) instead.
- If you see a `go.mod` file in the source try [vgo2nix](https://github.com/adisbladis/vgo2nix) instead.

## For Nixers - packaging Go applications

### Concept

`go2nix` provides an autmatic way to create Nix derivations for Go applications.

1. Start with app sources than can be built on your machine with `go build`.
   It means that you need to get all dependencies into current `GOPATH`.
2. Run `go2nix save` in application source dir where `main` package lives.
   This will create 2 files `default.nix` and `deps.nix` that can be moved
   into its own directory under `nixpkgs`.

### Example

If you are not sure how to organize your directory structure, [read this official guide](https://golang.org/doc/code.html) first.


Our project will be called `influxdb-demo` and this demo will be using the [influxdb client library](https://github.com/influxdata/influxdb/tree/master/client).

1. But first, prepare the project directory:

   ```sh
   mkdir example
   cd example
   mkdir bin pkg src
   ```

2. Change into a `shell` environment which contains GO and GIT:

   ```sh
   nix-shell -p go git go2nix
   ```

   **Note**: Make sure `go2nix` is at least version 1.1.1

3. Then set the `GOPATH`:

   ```sh
   export GOPATH=`pwd`
   mkdir -p src/github.com/qknight/influxdb-demo
   ```

4. Prepare `src/github.com/qknight/influxdb-demo/influxdb-client.go`:

   ```go
   package main

   import (
       "log"
       "time"

       "github.com/influxdata/influxdb/client/v2"
   )

   const (
       MyDB = "square_holes"
       username = "bubba"
       password = "bumblebeetuna"
   )

   func main() {
       // Make client
       c, err := client.NewHTTPClient(client.HTTPConfig{
           Addr: "http://localhost:8086",
           Username: username,
           Password: password,
       })

       if err != nil {
           log.Fatalln("Error: ", err)
       }

       // Create a new point batch
       bp, err := client.NewBatchPoints(client.BatchPointsConfig{
           Database:  MyDB,
           Precision: "s",
       })

       if err != nil {
           log.Fatalln("Error: ", err)
       }

       // Create a point and add to batch
       tags := map[string]string{"cpu": "cpu-total"}
       fields := map[string]interface{}{
           "idle":   10.1,
           "system": 53.3,
           "user":   46.6,
       }
       pt, err := client.NewPoint("cpu_usage", tags, fields, time.Now())

       if err != nil {
           log.Fatalln("Error: ", err)
       }

       bp.AddPoint(pt)

       // Write the batch
       c.Write(bp)
   }
   ```

   **Note**: We use [go influxdb client](https://github.com/influxdata/influxdb/tree/master/client) as an external library example.

5. Create a `GIT` repository

   ```sh
   cd src/github.com/qknight/influxdb-demo
   git init
   git add influxdb-client.go
   git commit -m 'initial commit'
   ```

   Also create the repository on github.com, in this example it would be `github.com/qknight/influxdb-demo`

   ```sh
   git remote add origin git@github.com:qknight/influxdb-demo.git
   git push -u origin master
   ```

   **Note:** `go2nix` requires a `GIT repository` to retrieve the `commit hash` and the `remote` so it can generate the values as `name`, `version`, `rev` and `goPackagePath` in the `default.nix` file.


6. Download the dependency the go-way

   ```sh
   go get
   ```

   **Note:** `go get` will populate `src/github.com/influxdata/influxdb` for you.

7. Building the project

   ```sh
   go build
   ```

8. Generate `default.nix`/`deps.nix`

   If `go` was able to build the binary, use `go2nix` to derive the `default.nix` and `deps.nix`:

   ```sh
   go2nix save
   ```

   **Note:** This gives you a `default.nix` and a `deps.nix` which can be put into `nixpkgs` but, if you want to use it with `nix-shell`, you need some tiny adaptions.

9. The `default.nix` file we generated

   ```nix
   # This file was generated by go2nix.
   { stdenv, buildGoPackage, fetchgit, fetchhg, fetchbzr, fetchsvn }:

   buildGoPackage rec {
     name = "influxdb-demo-${version}";
     version = "20161030-${stdenv.lib.strings.substring 0 7 rev}";
     rev = "718c85cd733bca964abf03f5371c939d19845f72";

     goPackagePath = "github.com/qknight/influxdb-demo";

     src = fetchgit {
       inherit rev;
       url = "git@github.com:qknight/influxdb-demo.git";
       sha256 = "0csbqcnncklimysgcbxlj190bynx1ppvyxvl5viz40fvbcj4l8xb";
     };

     goDeps = ./deps.nix;

     # TODO: add metadata https://nixos.org/nixpkgs/manual/#sec-standard-meta-attributes
     meta = {
     };
   }
   ```

   **Note:** Update the `fetchgit url` to use `https` instead of `ssh` or `nix-build` won't be able to fetch the software later.

   **Note:** If you want to use this `default.nix` in `nixpkgs` you should extend the [meta section](https://nixos.org/nixpkgs/manual/#chap-meta) and correct the `goPackagePath`.

10. The `deps.nix` we generated

    ```nix
    # This file was generated by go2nix.
    [
      {
        goPackagePath = "github.com/influxdata/influxdb";
        fetch = {
          type = "git";
          url = "https://github.com/influxdata/influxdb";
          rev = "bcb48a8ff2e9c118d5cfa04f03a62134b9c414c7";
          sha256 = "1f4jp7p2xxlsxhqbvglcl95y7fhab0w6zsvyqmqglmi0g1c5v21z";
        };
      }
    ]
    ```

11. Building the project using `nix-build`

    ```sh
    nix-build -E 'with import <nixpkgs> { };  callPackage ./default.nix {}'
    ```

    This will print something like this:

    ```
    these derivations will be built:
      /nix/store/8naxqswyhsmhqhjzskz5q0nf33nvyzsf-influxdb-demo-718c85c.drv
      /nix/store/p9bgjqn683xkmns36fpw2ipz5k57srl3-influxdb-bcb48a8.drv
      /nix/store/k0q9k2wfq0ccdsy04pyg4l01iqh5j1aa-go1.7-influxdb-demo-20161030-718c85c.drv
    building path(s) ‘/nix/store/45jjx5jakvlgwzq358pivr5x8xpgdsck-influxdb-bcb48a8’
    building path(s) ‘/nix/store/pqlqq2wjgixvf5j6qmrv5qyjh1bc6i8q-influxdb-demo-718c85c’
    exporting https://github.com/influxdata/influxdb (rev bcb48a8ff2e9c118d5cfa04f03a62134b9c414c7) into /nix/store/45jjx5jakvlgwzq358pivr5x8xpgdsck-influxdb-bcb48a8
    exporting https://github.com/qknight/influxdb-demo.git (rev 718c85cd733bca964abf03f5371c939d19845f72) into /nix/store/pqlqq2wjgixvf5j6qmrv5qyjh1bc6i8q-influxdb-demo-718c85c
    Initialized empty Git repository in /nix/store/45jjx5jakvlgwzq358pivr5x8xpgdsck-influxdb-bcb48a8/.git/
    Initialized empty Git repository in /nix/store/pqlqq2wjgixvf5j6qmrv5qyjh1bc6i8q-influxdb-demo-718c85c/.git/
    remote: Counting objects: 3, done.        
    remote: Compressing objects: 100% (2/2), done.        
    remote: Total 3 (delta 0), reused 3 (delta 0), pack-reused 0        
    From https://github.com/qknight/influxdb-demo
    * branch            HEAD       -> FETCH_HEAD
    Switched to a new branch 'fetchgit'
    removing `.git'...
    remote: Counting objects: 540, done.        
    remote: Compressing objects: 100% (506/506), done.        
    remote: Total 540 (delta 34), reused 179 (delta 6), pack-reused 0        
    Receiving objects: 100% (540/540), 1.44 MiB | 0 bytes/s, done.
    Resolving deltas: 100% (34/34), done.
    From https://github.com/influxdata/influxdb
    * branch            HEAD       -> FETCH_HEAD
    Switched to a new branch 'fetchgit'
    removing `.git'...
    building path(s) ‘/nix/store/9k4v7rhs5606fyia8mb341k71m3yrcbq-go1.7-influxdb-demo-20161030-718c85c-bin’, ‘/nix/store/av17zlfgnppl704wwxnh5pjpkcxac9k3-go1.7-influxdb-demo-20161030-718c85c’
    unpacking sources
    unpacking source archive /nix/store/pqlqq2wjgixvf5j6qmrv5qyjh1bc6i8q-influxdb-demo-718c85c
    source root is influxdb-demo-718c85c
    patching sources
    configuring
    grep: Invalid range end
    unpacking source archive /nix/store/45jjx5jakvlgwzq358pivr5x8xpgdsck-influxdb-bcb48a8
    building
    github.com/influxdata/influxdb/pkg/escape
    github.com/influxdata/influxdb/models
    github.com/influxdata/influxdb/client/v2
    github.com/qknight/influxdb-demo
    installing
    /tmp/nix-build-go1.7-influxdb-demo-20161030-718c85c.drv-0/go /tmp/nix-build-go1.7-influxdb-demo-20161030-718c85c.drv-0
    /tmp/nix-build-go1.7-influxdb-demo-20161030-718c85c.drv-0
    post-installation fixup
    shrinking RPATHs of ELF executables and libraries in /nix/store/9k4v7rhs5606fyia8mb341k71m3yrcbq-go1.7-influxdb-demo-20161030-718c85c-bin
    shrinking /nix/store/9k4v7rhs5606fyia8mb341k71m3yrcbq-go1.7-influxdb-demo-20161030-718c85c-bin/bin/influxdb-demo
    stripping (with flags -S) in /nix/store/9k4v7rhs5606fyia8mb341k71m3yrcbq-go1.7-influxdb-demo-20161030-718c85c-bin/bin 
    patching script interpreter paths in /nix/store/9k4v7rhs5606fyia8mb341k71m3yrcbq-go1.7-influxdb-demo-20161030-718c85c-bin
    shrinking RPATHs of ELF executables and libraries in /nix/store/av17zlfgnppl704wwxnh5pjpkcxac9k3-go1.7-influxdb-demo-20161030-718c85c
    patching script interpreter paths in /nix/store/av17zlfgnppl704wwxnh5pjpkcxac9k3-go1.7-influxdb-demo-20161030-718c85c
    /nix/store/9k4v7rhs5606fyia8mb341k71m3yrcbq-go1.7-influxdb-demo-20161030-718c85c-bin
    ```

    **Note**: The resulting binary will be in `/nix/store/9k4v7rhs5606fyia8mb341k71m3yrcbq-go1.7-influxdb-demo-20161030-718c85c-bin`.


12. Enabling `nix-shell`

    In order to use `nix-shell` you need to create a `shell.nix` file:
         
    ```nix
    let
      pkgs = import <nixpkgs> {};
    in pkgs.callPackage ./default.nix {}
    ```
    
    That will pass the derivations for `stdenv, buildGoPackage, fetchgit, fetchhg, fetchbzr, fetchsvn` into your package derivation. It will use `nixpkgs` (your current channel).


    After you updated the `default.nix` you can now use `nix-shell`:

    ```
    nix-shell
    these paths will be fetched (47.89 MiB download, 248.86 MiB unpacked):
      /nix/store/28wl3f34vfjpw0y5809bgr6382wqdscf-bash-4.3-p48
      /nix/store/7bdn99q835l042d8kcrn7yk61zkcqw6y-go1.7-govers-20150109-3b5f175
      /nix/store/aql31436x2fgr3rdj4fwbrgkvzw2kqf4-iana-etc-2.30
      /nix/store/bb2njjq32bh1wl2nl1zss0i8w1w2jgrz-tzdata-2016f
      /nix/store/dp2nf60lqzy1kbhd78ndf5nm3fb3qicd-gcc-wrapper-5.4.0
      /nix/store/drfmgb3kd3clxzi919mjrq5cnkw575w0-stdenv
      /nix/store/fkzyjrmrxv1zgswdq327h11ifbxdfpf9-go1.7-govers-20150109-3b5f175-bin
      /nix/store/g1pacy3pmx9r846z87xysfs29jz2b42q-parallel-20160722
      /nix/store/g4f1bms36hgg5abfd8xc4bj6sfzsy61d-bash-4.3-p48-info
      /nix/store/i76bwv541a7zj576hkdia62169r2nl0z-bash-4.3-p48-doc
      /nix/store/jqa9bh404crcxnyxa2lkffag7kw3yxw3-go-1.7.1
      /nix/store/jx5fyfd6mkpib1dmgy5rjirxff5vaczm-procps-3.3.11
      /nix/store/kx62kzlxcmzxgnxz1c95x7gln72iqd90-ncurses-6.0
      /nix/store/njn6il2fwaycjkd0jxbfj22gb25nd5df-perl-5.22.2
    fetching path ‘/nix/store/i76bwv541a7zj576hkdia62169r2nl0z-bash-4.3-p48-doc’...
    fetching path ‘/nix/store/g4f1bms36hgg5abfd8xc4bj6sfzsy61d-bash-4.3-p48-info’...
    fetching path ‘/nix/store/aql31436x2fgr3rdj4fwbrgkvzw2kqf4-iana-etc-2.30’...
    fetching path ‘/nix/store/bb2njjq32bh1wl2nl1zss0i8w1w2jgrz-tzdata-2016f’...
    ...
    ```

    After the dependencies were built you can now use: `go build` to build your software as usual!

    **Note:** You can also arrange a `default.nix` so that it can be used by `nixpkgs` and `nix-shell` but this is not covered here.


### Example Leaps

Some projects are built using `go get github.com/jeffail/leaps/cmd/...` and for these you need to identify the `src` directory to run `go2nix save` from. So here is another example:

1. `nix-shell -p go2nix git go`
2. `cd $(mktemp -d)`
3. `export GOPATH=$(pwd)`
4. `go get github.com/jeffail/leaps/cmd/...`
5. `cd src/github.com/jeffail/leaps/cmd/leaps`
6. `go2nix save`

**Note:** The resulting `default.nix` and `deps.nix` are created in the directory you are currently in.

# Installation

The preferred way of installing `go2nix` is to use `nix` like `nix-env -iA go2nix` or using it declaratively.

But you can also use `go get github.com/kamilchm/go2nix`.
