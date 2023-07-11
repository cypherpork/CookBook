# ElectrumX install on Apple Silicon Macos Ventura

I will use an ElectrumX fork from here: https://github.com/spesmilo/electrumx.git, because *"The original author dropped support for Bitcoin, which we intend to keep."* 

1.    Install brew.
      ```bash
      /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
      ```

2.    Do the post-install actions for brew:
      ```bash
      (echo; echo 'eval "$(/opt/homebrew/bin/brew shellenv)"') >> /Users/xroot/.zprofile
      eval "$(/opt/homebrew/bin/brew shellenv)"
      ```

      1.    Take care of `ulimit`:
      
      
      ```bash
      ulimit -n 10000
      echo "ulimit -n 10000" >> ~/.zprofile
      cat .zprofile > .bash_profile
      ```
      
3.    Brew install *NIX packages. Ensure at the end you run `/opt/homebrew/bin/python3`; if not - logout/login. If still not – fix your path.

      ``` bash
      brew install cmake gflags snappy lz4 daemontools python3 -y
      which python3
      ```

4.    ElectrumX runs only with old versions of RocksDB or FlatDB. In this instruction set, we will be **building** RocksDB 6.29.5:

      ```bash
      git clone --depth 1 --branch v6.29.5 https://github.com/facebook/rocksdb.git
      cd rocksdb
      nano CMakeLists.txt
      ```

      Find (`ctrl+w`) the string "`errors`" and comment out some code:

      ```cmake
      ...
        add_definitions(-DNPERF_CONTEXT)
      endif()
      
      option(FAIL_ON_WARNINGS "Treat compile warnings as errors" ON)
       if(FAIL_ON_WARNINGS)
         if(MSVC)
           set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /WX")
         else() # assume GCC
           set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror")
         endif()
      endif()
      
      option(WITH_ASAN "build with ASAN" OFF)
      if(WITH_ASAN)
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")
      ...
      ```
      
      After your edit, it should look like this:

      ```cmake
      ...
        add_definitions(-DNPERF_CONTEXT)
      endif()
      
      # option(FAIL_ON_WARNINGS "Treat compile warnings as errors" ON)
      # if(FAIL_ON_WARNINGS)
      #   if(MSVC)
      #     set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /WX")
      #   else() # assume GCC
      #     set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror")
      #   endif()
      # endif()
      
      option(WITH_ASAN "build with ASAN" OFF)
      if(WITH_ASAN)
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")
      ...
      ```
      Save, exit.
      
5. Make.

     ```bash
     mkdir build && cd build
     cmake ..
     ```

6. Get ready to compile.  Find out your versions for `snappy` and `lz4`. In my case:

     ```bash
     cd /opt/homebrew/Cellar/snappy/
     ls
     1.1.10
     cd /opt/homebrew/Cellar/lz4/
     ls
     1.9.4
     ```

7. Edit the line below – insert your versions from the previous step. Based on the versions that I have, my next command is:

     ```bash
     export LDFLAGS="-L/opt/homebrew/Cellar/snappy/1.1.10/lib -L/opt/homebrew/Cellar/lz4/1.9.4/lib"
     ```

8. And then:

      ```bash
      cd ~/rocksdb
      export CPPFLAGS=-I`pwd`/include
      export CPLUS_INCLUDE_PATH=${CPLUS_INCLUDE_PATH}${CPLUS_INCLUDE_PATH:+:}`pwd`/include/
      export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}${LD_LIBRARY_PATH:+:}`pwd`/build/
      export LIBRARY_PATH=${LIBRARY_PATH}${LIBRARY_PATH:+:}`pwd`/build/
      ```
      
9. Now you are ready to build. Get ready for a long-ish execution. Run:

      ```bash
      cd build 
      make
      ```


11. Softlink the freshly built library to where the further process will expect it:

     ```bash
     ln -s `pwd`/librocksdb.6.29.5.dylib /opt/homebrew/lib/librocksdb.6.29.5.dylib
     ln -s `pwd`/librocksdb.6.29.5.dylib /opt/homebrew/lib/librocksdb.6.dylib
     ln -s `pwd`/librocksdb.6.29.5.dylib /opt/homebrew/lib/librocksdb.dylib
     ```

12. Install some Python libraries:

     ```bash
     python3 -m pip install --upgrade pip aiohttp pylru aiorpcx ecdsa
     ```

13. Install the Python extension for RocksDB. This is your smoke test #1; if you get errors here, ask me:

     ```bash
     python3 -m pip install --upgrade --use-pep517 python-rocksdb
     ```

14. This is your smoke test #2:

     ```python
     python3
     >>> import rocksdb
     >>> db = rocksdb.DB("test.db", rocksdb.Options(create_if_missing=True))
     >>> db.put(b'a', b'data')
     >>> print(db.get(b'a'))
     b'data'
     >>> exit()
     ```

15. Download ElectrumX

     ```bash
     git clone https://github.com/spesmilo/electrumx.git
     ```

16. Edit `requirements.txt` to remove the line that says `plyvel` - you only need it for `LevelDB`, but you are using `RocksDB`. Save. Continue.

     ```bash
     cd electrumx
     nano requirements.txt
     ```

17. Install ElectrumX:

      ```bash
      pip3 install .
      ```

18. Make a directory where ElectrumX will be placing the database:

      ```bash
      cd
      mkdir electrumdb
      ```

19. Take care of the certificates. Respond to queries.  *The instructions I followed said that only FCDN is mandatory, but I discovered that, in my case, all prompts are optional. But then my server is not public.*

      ```bash
      mkdir .electrumx
      cd .electrumx
      openssl genrsa -out server.key 2048
      openssl req -new -key server.key -out server.csr
      openssl x509 -req -days 1825 -in server.csr -signkey server.key -out server.crt
      ```

20. It's time to set up the variables configuring your ElectrumX server. It is done within the ENV variables within `~/scripts/electrumx/env` folder. Each variable is located in a separate file. Refer [here](https://electrumx-spesmilo.readthedocs.io/en/latest/environment.html) for the full list of options. Here's my setup:

      ```bash
      bash$ cd ~/scripts/electrumx/env/
      bash$ ls -la
      total 104
      drwxr-xr-x  15 electrumuser  staff  480 Jul  9 03:27 .
      drwxr-xr-x   7 electrumuser  staff  224 Jul  9 03:13 ..
      -rw-r--r--@  1 electrumuser  staff    8 Jul  5 22:21 COIN
      -rw-r--r--@  1 electrumuser  staff   37 Jul  9 03:13 DAEMON_URL
      -rw-r--r--@  1 electrumuser  staff   23 Jul  5 22:47 DB_DIRECTORY
      -rw-r--r--@  1 electrumuser  staff    7 Jul  6 23:21 DB_ENGINE
      -rw-r--r--@  1 electrumuser  staff   39 Jul  5 22:52 ELECTRUMX
      -rw-r--r--@  1 electrumuser  staff    3 Jul  5 22:59 INITIAL_CONCURRENT
      -rw-r--r--@  1 electrumuser  staff    5 Jul  5 23:00 MAX_SESSIONS
      -rw-r--r--@  1 electrumuser  staff    7 Jul  9 03:27 NET
      -rw-r--r--@  1 electrumuser  staff   23 Jul  9 03:14 SERVICES
      -rw-r--r--@  1 electrumuser  staff   34 Jul  5 22:56 SSL_CERTFILE
      -rw-r--r--@  1 electrumuser  staff   34 Jul  5 22:57 SSL_KEYFILE
      -rw-r--r--@  1 electrumuser  staff    6 Jul  5 22:54 USERNAME
      bash$ cat COIN
      Bitcoin
      bash$ cat NET
      signet
      bash$ cat DB_ENGINE
      rocksdb
      bash$
      ```

21. Set up logging: create a directory to place logs, and edit the `run` file so logs are written to the directory you just created:

      ```bash
      mkdir electrumXlogs
      nano ~/scripts/electrumx/log/run
      ```

22. It's time to hook up ElectrumX as a service. We will be using `daemontools`; more on these [here](https://cr.yp.to/daemontools.html). 

      ```bash
      cd
      mkdir ~/service
      mkdir -p ~/scripts/electrumx
      cp -R ~/electrumx/contrib/daemontools/ ~/scripts/electrumx
      sudo brew services start daemontools 
      # Services are stored in /opt/homebrew/etc/service/
      svscan ~/service & disown
      cd ~/service
      ln -s ~/scripts/electrumx electrumx
      ```

23. You should now see how your ElectrumX is synchronizing:

     ``` bash
     tail -F ~/electrumXlogs/current | tai64nlocal
     ```

24. Minimal `daemontools` cheat sheet:

     *    To stop the service: `svc -d ~/service/electrumx`
     *    To start the service: `svc -u ~/service/electrumx`
     *    To get the service status: `svstat ~/service/electrumx`
