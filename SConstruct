
# build file for 10gen db
# this request scons
# you can get from http://www.scons.org
# then just type scons

# some common tasks
#   build 64-bit mac and pushing to s3
#      scons --64 s3dist
#      scons --distname=0.8 s3dist
#      all s3 pushes require settings.py and simples3

import os
import sys
import types
import re
import shutil
import urllib
import urllib2

# --- options ----
AddOption('--prefix',
          dest='prefix',
          type='string',
          nargs=1,
          action='store',
          metavar='DIR',
          help='installation prefix')

AddOption('--distname',
          dest='distname',
          type='string',
          nargs=1,
          action='store',
          metavar='DIR',
          help='dist name (0.8.0)')


AddOption( "--64",
           dest="force64",
           type="string",
           nargs=0,
           action="store",
           help="whether to force 64 bit" )


AddOption( "--32",
           dest="force32",
           type="string",
           nargs=0,
           action="store",
           help="whether to force 32 bit" )


AddOption( "--mm",
           dest="mm",
           type="string",
           nargs=0,
           action="store",
           help="use main memory instead of memory mapped files" )


AddOption( "--release",
           dest="release",
           type="string",
           nargs=0,
           action="store",
           help="relase build")

AddOption('--java',
          dest='javaHome',
          type='string',
          default="/opt/java/",
          nargs=1,
          action='store',
          metavar='DIR',
          help='java home')

AddOption('--nojni',
          dest='nojni',
          type="string",
          nargs=0,
          action="store",
          help="turn off jni support" )


AddOption('--usesm',
          dest='usesm',
          type="string",
          nargs=0,
          action="store",
          help="use spider monkey for javascript" )

AddOption('--usejvm',
          dest='usejvm',
          type="string",
          nargs=0,
          action="store",
          help="use java for javascript" )

AddOption( "--v8" ,
           dest="v8home",
           type="string",
           default="../v8/",
           nargs=1,
           action="store",
           metavar="dir",
           help="v8 location")

AddOption( "--d",
           dest="debugBuild",
           type="string",
           nargs=0,
           action="store",
           help="debug build no optimization, etc..." )

AddOption( "--dd",
           dest="debugBuildAndLogging",
           type="string",
           nargs=0,
           action="store",
           help="debug build no optimization, additional debug logging, etc..." )

AddOption( "--recstore",
           dest="recstore",
           type="string",
           nargs=0,
           action="store",
           help="use new recstore" )

AddOption( "--noshell",
           dest="noshell",
           type="string",
           nargs=0,
           action="store",
           help="don't build shell" )

# --- environment setup ---

def printLocalInfo():
    import sys, SCons
    print( "scons version: " + SCons.__version__ )
    print( "python version: " + " ".join( [ `i` for i in sys.version_info ] ) )

printLocalInfo()

env = Environment()

if GetOption( "recstore" ) != None:
    env.Append( CPPDEFINES=[ "_RECSTORE" ] )
env.Append( CPPDEFINES=[ "_SCONS" ] )
env.Append( CPPPATH=[ "." ] )

boostLibs = [ "thread" , "filesystem" , "program_options" ]

onlyServer = len( COMMAND_LINE_TARGETS ) == 0 or ( len( COMMAND_LINE_TARGETS ) == 1 and str( COMMAND_LINE_TARGETS[0] ) in [ "mongod" , "mongos" , "test" ] )
nix = False
useJavaHome = False
linux = False
linux64  = False
darwin = False
windows = False
force64 = not GetOption( "force64" ) is None
force32 = not GetOption( "force32" ) is None
release = not GetOption( "release" ) is None

debugBuild = ( not GetOption( "debugBuild" ) is None ) or ( not GetOption( "debugBuildAndLogging" ) is None )
debugLogging = not GetOption( "debugBuildAndLogging" ) is None
noshell = not GetOption( "noshell" ) is None
nojni = not GetOption( "nojni" ) is None

usesm = not GetOption( "usesm" ) is None
usejvm = not GetOption( "usejvm" ) is None

if ( usesm and usejvm ):
    print( "can't say usesm and usejvm at the same time" )
    Exit(1)

if ( not ( usesm or usejvm ) ):
    usesm = True

# ------    SOURCE FILE SETUP -----------

commonFiles = Split( "stdafx.cpp buildinfo.cpp db/jsobj.cpp db/json.cpp db/commands.cpp db/lasterror.cpp db/nonce.cpp db/queryutil.cpp shell/mongo.cpp" )
commonFiles += [ "util/background.cpp" , "util/mmap.cpp" ,  "util/sock.cpp" ,  "util/util.cpp" , "util/message.cpp" ]
commonFiles += Glob( "util/*.c" )
commonFiles += Split( "client/connpool.cpp client/dbclient.cpp client/model.cpp" )
commonFiles += [ "scripting/engine.cpp" ]

#mmap stuff

if GetOption( "mm" ) != None:
    commonFiles += [ "util/mmap_mm.cpp" ]
elif os.sys.platform == "win32":
    commonFiles += [ "util/mmap_win.cpp" ]
else:
    commonFiles += [ "util/mmap_posix.cpp" ]

if os.path.exists( "util/processinfo_" + os.sys.platform + ".cpp" ):
    commonFiles += [ "util/processinfo_" + os.sys.platform + ".cpp" ]
else:
    commonFiles += [ "util/processinfo_none.cpp" ]

coreDbFiles = []
coreServerFiles = [ "util/message_server_port.cpp" , "util/message_server_asio.cpp" ]

serverOnlyFiles = Split( "db/query.cpp db/introspect.cpp db/btree.cpp db/clientcursor.cpp db/tests.cpp db/repl.cpp db/btreecursor.cpp db/cloner.cpp db/namespace.cpp db/matcher.cpp db/dbcommands.cpp db/dbeval.cpp db/dbwebserver.cpp db/dbinfo.cpp db/dbhelpers.cpp db/instance.cpp db/pdfile.cpp db/cursor.cpp db/security_commands.cpp db/security.cpp util/miniwebserver.cpp db/storage.cpp db/reccache.cpp db/queryoptimizer.cpp" )

if usesm:
    commonFiles += [ "scripting/engine_spidermonkey.cpp" ]
    nojni = True
elif not nojni:
    commonFiles += [ "scripting/engine_java.cpp" ]
else:
    commonFiles += [ "scripting/engine_none.cpp" ]
    nojni = True

coreShardFiles = []
shardServerFiles = coreShardFiles + Glob( "s/strategy*.cpp" ) + [ "s/commands_admin.cpp" , "s/commands_public.cpp" , "s/request.cpp" ,  "s/cursors.cpp" ,  "s/server.cpp" ] + [ "s/shard.cpp" , "s/shardkey.cpp" , "s/config.cpp" ]
serverOnlyFiles += coreShardFiles + [ "s/d_logic.cpp" ]

allClientFiles = commonFiles + coreDbFiles + [ "client/clientOnly.cpp" , "client/gridfs.cpp" ];

# ---- other build setup -----

platform = os.sys.platform
if "uname" in dir(os):
    processor = os.uname()[4]
else:
    processor = "i386"

if force32:
    processor = "i386"
if force64:
    processor = "x86_64"

DEFAULT_INSTALl_DIR = "/usr/local"
installDir = DEFAULT_INSTALl_DIR
nixLibPrefix = "lib"

distName = GetOption( "distname" )

javaHome = GetOption( "javaHome" )
javaVersion = "i386";
javaLibs = []

distBuild = len( COMMAND_LINE_TARGETS ) == 1 and ( str( COMMAND_LINE_TARGETS[0] ) == "s3dist" or str( COMMAND_LINE_TARGETS[0] ) == "dist" )
if distBuild:
    release = True

if GetOption( "prefix" ):
    installDir = GetOption( "prefix" )

def findVersion( root , choices ):
    for c in choices:
        if ( os.path.exists( root + c ) ):
            return root + c
    raise "can't find a version of [" + root + "] choices: " + choices

def choosePathExist( choices , default=None):
    for c in choices:
        if c != None and os.path.exists( c ):
            return c
    return default

if "darwin" == os.sys.platform:
    darwin = True
    platform = "osx" # prettier than darwin

    env.Append( CPPPATH=[ "-I/System/Library/Frameworks/JavaVM.framework/Versions/CurrentJDK/Headers/" ] )

    env.Append( CPPFLAGS=" -mmacosx-version-min=10.4 " )
    if not nojni:
        env.Append( FRAMEWORKS=["JavaVM"] )

    if os.path.exists( "/usr/bin/g++-4.2" ):
        env["CXX"] = "g++-4.2"

    nix = True

    if force64:
        env.Append( CPPPATH=["/usr/64/include"] )
        env.Append( LIBPATH=["/usr/64/lib"] )
        if installDir == DEFAULT_INSTALl_DIR and not distBuild:
            installDir = "/usr/64/"
    else:
        env.Append( CPPPATH=[ "/sw/include" , "/opt/local/include"] )
        env.Append( LIBPATH=["/sw/lib/", "/opt/local/lib"] )

elif "linux2" == os.sys.platform:
    linux = True
    useJavaHome = True
    javaOS = "linux"
    platform = "linux"

    javaHome = choosePathExist( [ javaHome , "/usr/lib/jvm/java/" , os.environ.get( "JAVA_HOME" ) ] , "/usr/lib/jvm/java/" )

    if os.uname()[4] == "x86_64" and not force32:
        linux64 = True
        javaVersion = "amd64"
        nixLibPrefix = "lib64"
        env.Append( LIBPATH=["/usr/lib64" , "/lib64" ] )
        env.Append( LIBS=["pthread"] )

    if force32:
        env.Append( LIBPATH=["/usr/lib32"] )

    nix = True

elif "sunos5" == os.sys.platform:
     nix = True
     useJavaHome = True
     javaHome = "/usr/lib/jvm/java-6-sun/"
     javaOS = "solaris"
     env.Append( CPPDEFINES=[ "__linux__" , "__sunos__" ] )
     env.Append( LIBS=["socket"] )

elif "win32" == os.sys.platform:
    windows = True
    boostDir = "C:/Program Files/Boost/boost_1_35_0"

    if not os.path.exists( boostDir ):
        print( "can't find boost" )
        Exit(1)

    boostLibs = []

    javaHome = findVersion( "C:/Program Files/java/" ,
                            [ "jdk" , "jdk1.6.0_10" ] )
    winSDKHome = findVersion( "C:/Program Files/Microsoft SDKs/Windows/" ,
                              [ "v6.0" , "v6.0a" , "v6.1" ] )

    env.Append( CPPPATH=[ boostDir , javaHome + "/include" , javaHome + "/include/win32" , "pcre-7.4" , winSDKHome + "/Include" ] )

    env.Append( CPPFLAGS=" /EHsc /W3 " )
    env.Append( CPPDEFINES=["WIN32","_CONSOLE","_CRT_SECURE_NO_WARNINGS","HAVE_CONFIG_H","PCRE_STATIC","_UNICODE","UNICODE" ] )

    if release:
        env.Append( CPPDEFINES=[ "NDEBUG" ] )
        env.Append( CPPFLAGS= " /O2 /Oi /GL /FD /MT /Gy /nologo /Zi /TP /errorReport:prompt /Gm " )
        # /Yu"stdafx.h" /Fp"Release\db.pch" /Fo"Release\\" /Fd"Release\vc90.pdb"
    else:
        env.Append( CPPDEFINES=[ "_DEBUG" ] )
        env.Append( CPPFLAGS=" /Od /Gm /RTC1 /MDd /ZI " )
        # /Fo"Debug\\" /Fd"Debug\vc90.pdb"

    env.Append( LIBPATH=[ boostDir + "/Lib" , javaHome + "/Lib" , winSDKHome + "/Lib" ] )
    javaLibs += [ "jvm" ];

    def pcreFilter(x):
        name = x.name
        if x.name.endswith( "dftables.c" ):
            return False
        if x.name.endswith( "pcredemo.c" ):
            return False
        if x.name.endswith( "pcretest.c" ):
            return False
        if x.name.endswith( "unittest.cc" ):
            return False
        if x.name.endswith( "pcregrep.c" ):
            return False
        return True

    pcreFiles = []
    pcreFiles += filter( pcreFilter , Glob( "pcre-7.4/*.c"  ) )
    pcreFiles += filter( pcreFilter , Glob( "pcre-7.4/*.cc" ) )
    commonFiles += pcreFiles
    allClientFiles += pcreFiles

    env.Append( LIBS=Split("ws2_32.lib kernel32.lib user32.lib gdi32.lib winspool.lib comdlg32.lib advapi32.lib shell32.lib ole32.lib oleaut32.lib uuid.lib odbc32.lib odbccp32.lib" ) )

else:
    print( "No special config for [" + os.sys.platform + "] which probably means it won't work" )

if useJavaHome:
    env.Append( CPPPATH=[ javaHome + "include" , javaHome + "include/" + javaOS ] )
    env.Append( LIBPATH=[ javaHome + "jre/lib/" + javaVersion + "/server" , javaHome + "jre/lib/" + javaVersion ] )

    if not nojni:
        javaLibs += [ "java" , "jvm" ]

    env.Append( LINKFLAGS="-Xlinker -rpath -Xlinker " + javaHome + "jre/lib/" + javaVersion + "/server" )
    env.Append( LINKFLAGS="-Xlinker -rpath -Xlinker " + javaHome + "jre/lib/" + javaVersion  )

if nix:
    env.Append( CPPFLAGS="-fPIC -fno-strict-aliasing -ggdb -pthread -Wall -Wsign-compare -Wno-unknown-pragmas" )
    env.Append( CXXFLAGS=" -Wnon-virtual-dtor " )
    env.Append( LINKFLAGS=" -fPIC -pthread " )
    env.Append( LIBS=[] )

    if debugBuild:
        env.Append( CPPFLAGS=" -O0 -fstack-protector -fstack-check" );
    else:
        env.Append( CPPFLAGS=" -O3" )

    if debugLogging:
        env.Append( CPPFLAGS=" -D_DEBUG" );

    if force64:
        env.Append( CFLAGS="-m64" )
        env.Append( CXXFLAGS="-m64" )
        env.Append( LINKFLAGS="-m64" )

    if force32:
        env.Append( CFLAGS="-m32" )
        env.Append( CXXFLAGS="-m32" )
        env.Append( LINKFLAGS="-m32" )


# --- check system ---

def getGitVersion():
    if not os.path.exists( ".git" ):
        return "nogitversion"

    version = open( ".git/HEAD" ,'r' ).read().strip()
    if not version.startswith( "ref: " ):
        return version
    version = version[5:]
    f = ".git/" + version
    if not os.path.exists( f ):
        return version
    return open( f , 'r' ).read().strip()

def getSysInfo():
    import os, sys
    if windows:
        return "windows " + str( sys.getwindowsversion() )
    else:
        return " ".join( os.uname() )

def setupBuildInfoFile( outFile ):
    version = getGitVersion()
    sysInfo = getSysInfo()
    contents = "#include \"stdafx.h\"\n"
    contents += "#include <iostream>\n"
    contents += "namespace mongo { const char * gitVersion(){ return \"" + version + "\"; } }\n"
    contents += "namespace mongo { const char * sysInfo(){ return \"" + sysInfo + "\"; } }\n"

    if os.path.exists( outFile ) and open( outFile ).read().strip() == contents.strip():
        return

    out = open( outFile , 'w' )
    out.write( contents )
    out.close()

setupBuildInfoFile( "buildinfo.cpp" )

def bigLibString( myenv ):
    s = str( myenv["LIBS"] )
    if 'SLIBS' in myenv._dict:
        s += str( myenv["SLIBS"] )
    return s
    

def doConfigure( myenv , needJava=True , needPcre=True , shell=False ):
    conf = Configure(myenv)
    myenv["LINKFLAGS_CLEAN"] = list( myenv["LINKFLAGS"] )
    myenv["LIBS_CLEAN"] = list( myenv["LIBS"] )

    if nix and not shell:
        if not conf.CheckLib( "stdc++" ):
            print( "can't find stdc++ library which is needed" );
            Exit(1)

    def myCheckLib( poss , failIfNotFound=False , java=False , staticOnly=False):

        if type( poss ) != types.ListType :
            poss = [poss]

        allPlaces = [];
        if nix and release:
            allPlaces += myenv["LIBPATH"]
            if not force64:
                allPlaces += [ "/usr/lib" , "/usr/local/lib" ]

            for p in poss:
                for loc in allPlaces:
                    fullPath = loc + "/lib" + p + ".a"
                    if os.path.exists( fullPath ):
                        myenv['_LIBFLAGS']='${_stripixes(LIBLINKPREFIX, LIBS, LIBLINKSUFFIX, LIBPREFIXES, LIBSUFFIXES, __env__)} $SLIBS'
                        myenv.Append( SLIBS=" " + fullPath + " " )
                        return True


        if release and not java and failIfNotFound:
            extra = ""
            if linux64 and shell:
                extra += " 32 bit version for shell"
            print( "ERROR: can't find static version of: " + str( poss ) + extra + " in: " + str( allPlaces ) )
            Exit(1)

        res = not staticOnly and conf.CheckLib( poss )
        if res:
            return True

        if failIfNotFound:
            print( "can't find " + str( poss ) + " in " + str( myenv["LIBPATH"] ) )
            Exit(1)

        return False

    if needPcre and not conf.CheckCXXHeader( 'pcrecpp.h' ):
        print( "can't find pcre" )
        Exit(1)

    if not conf.CheckCXXHeader( "boost/filesystem/operations.hpp" ):
        print( "can't find boost headers" )
        if shell:
            print( "\tshell might not compile" )
        else:
            Exit(1)

    if conf.CheckCXXHeader( "boost/asio.hpp" ):
        # TODO: turn this back on when ASIO working
        myenv.Append( CPPDEFINES=[ "USE_ASIO_OFF" ] )
    else:
        print( "WARNING: old version of boost - you should consider upgrading" )

    for b in boostLibs:
        l = "boost_" + b
        myCheckLib( [ l + "-mt" , l ] , release or not shell)

    if needJava:
        for j in javaLibs:
            myCheckLib( j , True , True )

    if nix and needPcre:
        myCheckLib( "pcrecpp" , True )
        myCheckLib( "pcre" , True )
        
    myenv["_HAVEPCAP"] = myCheckLib( "pcap", staticOnly=release )

    # this is outside of usesm block so don't have to rebuild for java
    if windows:
        myenv.Append( CPPDEFINES=[ "XP_WIN" ] )
    else:
        myenv.Append( CPPDEFINES=[ "XP_UNIX" ] )

    if usesm:

        myCheckLib( [ "js" , "mozjs" ] , True )
        mozHeader = "js"
        if bigLibString(myenv).find( "mozjs" ) >= 0:
            myenv.Append( CPPDEFINES=[ "MOZJS" ] )
            mozHeader = "mozjs"

        if not conf.CheckHeader( mozHeader + "/jsapi.h" ):
            if conf.CheckHeader( "jsapi.h" ):
                myenv.Append( CPPDEFINES=[ "OLDJS" ] )
                print( "warning: old spider monkey version" )
            else:
                print( "no spider monkey headers!" )
                Exit(1)

    if shell:
        haveReadLine = False
        if darwin:
            myenv.Append( CPPDEFINES=[ "USE_READLINE" ] )
            if force64:
                myCheckLib( "readline" , True )
                myCheckLib( "ncurses" , True )
            else:
                myenv.Append( LINKFLAGS=" /usr/lib/libreadline.dylib " )
        elif myCheckLib( "readline" ):
            myenv.Append( CPPDEFINES=[ "USE_READLINE" ] )
            myCheckLib( "tinfo" , staticOnly=release )
        else:
            print( "WARNING: no readline, shell will be a bit ugly" )

        if linux:
            myCheckLib( "rt" , True )

    # this will add it iff it exists and works
    myCheckLib( "boost_system-mt" )

    return conf.Finish()

env = doConfigure( env )

# --- js concat ---

def concatjs(target, source, env):

    outFile = str( target[0] )

    fullSource = ""

    for s in source:
        f = open( str(s) , 'r' )
        for l in f:
            fullSource += l

    out = open( outFile , 'w' )
    out.write( fullSource )

    return None

jsBuilder = Builder(action = concatjs,
                    suffix = '.jsall',
                    src_suffix = '.js')

env.Append( BUILDERS={'JSConcat' : jsBuilder})

# --- jsh ---

def jsToH(target, source, env):

    outFile = str( target[0] )
    if len( source ) != 1:
        raise Exception( "wrong" )

    h = "const char * jsconcatcode = \n"

    for l in open( str(source[0]) , 'r' ):
        l = l.strip()
        l = l.partition( "//" )[0]
        l = l.replace( '\\' , "\\\\" )
        l = l.replace( '"' , "\\\"" )


        h += '"' + l + "\\n\"\n "

    h += ";\n\n"

    out = open( outFile , 'w' )
    out.write( h )

    return None

jshBuilder = Builder(action = jsToH,
                    suffix = '.cpp',
                    src_suffix = '.jsall')

env.Append( BUILDERS={'JSHeader' : jshBuilder})


# --- targets ----

def removeIfInList( lst , thing ):
    if thing in lst:
        lst.remove( thing )

clientEnv = env.Clone();
clientEnv.Append( CPPPATH=["../"] )
clientEnv.Prepend( LIBS=[ "libmongoclient.a"] )
clientEnv.Append( LIBPATH=["."] )
l = clientEnv[ "LIBS" ]
removeIfInList( l , "pcre" )
removeIfInList( l , "pcrecpp" )

testEnv = env.Clone()
testEnv.Append( CPPPATH=["../"] )
testEnv.Prepend( LIBS=[ "libmongotestfiles.a" , "unittest" ] )
testEnv.Append( LIBPATH=["."] )


# ----- TARGETS ------


# main db target
mongod = env.Program( "mongod" , commonFiles + coreDbFiles + serverOnlyFiles + [ "db/db.cpp" ]  )
Default( mongod )

# tools
allToolFiles = commonFiles + coreDbFiles + serverOnlyFiles + [ "client/gridfs.cpp", "tools/Tool.cpp" ]
env.Program( "mongodump" , allToolFiles + [ "tools/dump.cpp" ] )
env.Program( "mongorestore" , allToolFiles + [ "tools/restore.cpp" ] )

env.Program( "mongoexport" , allToolFiles + [ "tools/export.cpp" ] )
env.Program( "mongoimportjson" , allToolFiles + [ "tools/importJSON.cpp" ] )

env.Program( "mongofiles" , allToolFiles + [ "tools/files.cpp" ] )

env.Program( "mongobridge" , allToolFiles + [ "tools/bridge.cpp" ] )

# mongos
mongos = env.Program( "mongos" , commonFiles + coreDbFiles + coreServerFiles + shardServerFiles )

# c++ library
clientLibName = str( env.Library( "mongoclient" , allClientFiles )[0] )
env.Library( "mongotestfiles" , commonFiles + coreDbFiles + serverOnlyFiles )

clientTests = []

# examples
clientTests += [ clientEnv.Program( "firstExample" , [ "client/examples/first.cpp" ] ) ]
clientTests += [ clientEnv.Program( "secondExample" , [ "client/examples/second.cpp" ] ) ]
clientTests += [ clientEnv.Program( "whereExample" , [ "client/examples/whereExample.cpp" ] ) ]
clientTests += [ clientEnv.Program( "authTest" , [ "client/examples/authTest.cpp" ] ) ]

# testing
test = testEnv.Program( "test" , Glob( "dbtests/*.cpp" ) )
perftest = testEnv.Program( "perftest", "dbtests/perf/perftest.cpp" )
clientTests += [ clientEnv.Program( "clientTest" , [ "client/examples/clientTest.cpp" ] ) ]

# --- sniffer ---
if darwin or clientEnv["_HAVEPCAP"]:
    sniffEnv = clientEnv.Clone()
    sniffEnv.Append( LIBS=[ "pcap" ] )
    sniffEnv.Program( "mongosniff" , "tools/sniffer.cpp" )

# --- shell ---

env.JSConcat( "shell/mongo.jsall"  , Glob( "shell/*.js" ) )
env.JSHeader( "shell/mongo.jsall" )

shellEnv = env.Clone();

if release and ( ( darwin and force64 ) or linux64 ):
    shellEnv["LINKFLAGS"] = env["LINKFLAGS_CLEAN"]
    shellEnv["LIBS"] = env["LIBS_CLEAN"]
    shellEnv["SLIBS"] = ""

if noshell:
    print( "not building shell" )
elif not onlyServer:
    weird = force64

    if force64:
        shellEnv["CFLAGS"].remove("-m64")
        shellEnv["CXXFLAGS"].remove("-m64")
        shellEnv["LINKFLAGS"].remove("-m64")
        shellEnv["CPPPATH"].remove( "/usr/64/include" )
        shellEnv["LIBPATH"].remove( "/usr/64/lib" )
        shellEnv.Append( CPPPATH=[ "/sw/include" , "/opt/local/include"] )
        shellEnv.Append( LIBPATH=[ "/sw/lib/", "/opt/local/lib" , "/usr/lib" ] )

    l = shellEnv["LIBS"]
    if linux64:
        removeIfInList( l , "java" )
        removeIfInList( l , "jvm" )

    removeIfInList( l , "pcre" )
    removeIfInList( l , "pcrecpp" )

    if windows:
        shellEnv.Append( LIBS=["winmm.lib"] )

    coreShellFiles = [ "shell/dbshell.cpp" , "shell/utils.cpp" ]

    if weird:
        shell32BitFiles = coreShellFiles
        for f in allClientFiles:
            shell32BitFiles.append( "32bit/" + str( f ) )

        shellEnv.VariantDir( "32bit" , "." )
    else:
        shellEnv.Append( LIBPATH=[ "." ] )

    shellEnv = doConfigure( shellEnv , needPcre=False , needJava=False , shell=True )

    if weird:
        mongo = shellEnv.Program( "mongo" , shell32BitFiles )
    else:
        shellEnv.Append( LIBS=[ "mongoclient"] )
        mongo = shellEnv.Program( "mongo" , coreShellFiles )


#  ---- RUNNING TESTS ----

testEnv.Alias( "dummySmokeSideEffect", [], [] )

def addSmoketest( name, deps, actions ):
    testEnv.Alias( name, deps, actions )
    testEnv.AlwaysBuild( name )
    # Prevent smoke tests from running in parallel
    testEnv.SideEffect( "dummySmokeSideEffect", name )

def ensureDir( name ):
    d = os.path.dirname( name )
    if not os.path.exists( d ):
        print( "Creating dir: " + name );
        os.makedirs( d )
        if not os.path.exists( d ):
            print( "Failed to create dir: " + name );
            Exit( 1 )

def testSetup( env , target , source ):
    ensureDir( "/tmp/unittest/" )

addSmoketest( "smoke", [ "test" ] , [ testSetup , test[ 0 ].abspath ] )
addSmoketest( "smokePerf", [ "perftest" ] , [ perftest[ 0 ].abspath ] )

clientExec = [ x[0].abspath for x in clientTests ]
def runClientTests( env, target, source ):
    global clientExec
    global mongodForTestsPort
    import subprocess
    for i in clientExec:
        if subprocess.call( [ i, "--port", mongodForTestsPort ] ) != 0:
            return True
    if subprocess.Popen( [ mongod[0].abspath, "msg", "ping", mongodForTestsPort ], stdout=subprocess.PIPE ).communicate()[ 0 ].count( "****ok" ) == 0:
        return True
    if subprocess.call( [ mongod[0].abspath, "msg", "ping", mongodForTestsPort ] ) != 0:
        return True
    return False
addSmoketest( "smokeClient" , clientExec, runClientTests )
addSmoketest( "mongosTest" , [ mongos[0].abspath ] , [ mongos[0].abspath + " --test" ] )

def jsSpec( suffix ):
    import os.path
    args = [ os.path.dirname( mongo[0].abspath ), "jstests" ] + suffix
    return apply( os.path.join, args )

def jsDirTestSpec( dir ):
    return mongo[0].abspath + " --nodb " + jsSpec( [ dir, "*.js" ] )

def runShellTest( env, target, source ):
    global mongodForTestsPort
    import subprocess
    target = str( target[0] )
    if target == "smokeJs":
        spec = jsSpec( [ "_runner.js" ] )
    elif target == "smokeQuota":
        spec = jsSpec( [ "quota", "quota1.js" ] )
    else:
        print( "invalid target for runShellTest()" )
        Exit( 1 )
    return subprocess.call( [ mongo[0].abspath, "--port", mongodForTestsPort, spec ] )

# These tests require the mongo shell
if not onlyServer and not noshell:
    addSmoketest( "smokeJs", [ "mongo" ], runShellTest )
    addSmoketest( "smokeClone", [ "mongo", "mongod" ], [ jsDirTestSpec( "clone" ) ] )
    addSmoketest( "smokeRepl", [ "mongo", "mongod" ], [ jsDirTestSpec( "repl" ) ] )
    addSmoketest( "smokeDisk", [ "mongo", "mongod" ], [ jsDirTestSpec( "disk" ) ] )
    addSmoketest( "smokeSharding", [ "mongo", "mongod", "mongos" ], [ jsDirTestSpec( "sharding" ) ] )
    addSmoketest( "smokeJsPerf", [ "mongo" ], [ mongo[0].abspath + " " + jsSpec( [ "perf", "*.js" ] ) ] )
    addSmoketest( "smokeQuota", [ "mongo" ], runShellTest )

mongodForTests = None
mongodForTestsPort = "27017"

def startMongodForTests( env, target, source ):
    global mongodForTests
    global mongodForTestsPort
    global mongod
    if mongodForTests:
        return
    mongodForTestsPort = "40000"
    import os
    dirName = "/data/db/sconsTests/"
    ensureDir( dirName )
    from subprocess import Popen
    mongodForTests = Popen( [ mongod[0].abspath, "--port", mongodForTestsPort, "--dbpath", dirName ] )
    # Wait for mongod to start
    import time
    time.sleep( 2 )
    if mongodForTests.poll() is not None:
        print( "Failed to start mongod" )
        mongodForTests = None
        Exit( 1 )

def stopMongodForTests():
    global mongodForTests
    if not mongodForTests:
        return
    if mongodForTests.poll() is not None:
        print( "Failed to start mongod" )
        mongodForTests = None
        Exit( 1 )
    try:
        # This function not available in Python 2.5
        mongodForTests.terminate()
    except AttributeError:
        from os import kill
        kill( mongodForTests.pid, 15 )
    mongodForTests.wait()

testEnv.Alias( "startMongod", ["mongod"], [startMongodForTests] );
testEnv.AlwaysBuild( "startMongod" );
testEnv.SideEffect( "dummySmokeSideEffect", "startMongod" )

def addMongodReqTargets( env, target, source ):
    mongodReqTargets = [ "smokeClient", "smokeJs", "smokeQuota" ]
    for target in mongodReqTargets:
        testEnv.Depends( target, "startMongod" )
        testEnv.Depends( "smokeAll", target )

testEnv.Alias( "addMongodReqTargets", [], [addMongodReqTargets] )
testEnv.AlwaysBuild( "addMongodReqTargets" )

testEnv.Alias( "smokeAll", [ "smoke", "mongosTest", "smokeClone", "smokeRepl", "addMongodReqTargets", "smokeDisk", "smokeSharding" ] )
testEnv.AlwaysBuild( "smokeAll" )

def addMongodReqNoJsTargets( env, target, source ):
    mongodReqTargets = [ "smokeClient" ]
    for target in mongodReqTargets:
        testEnv.Depends( target, "startMongod" )
        testEnv.Depends( "smokeAllNoJs", target )

testEnv.Alias( "addMongodReqNoJsTargets", [], [addMongodReqNoJsTargets] )
testEnv.AlwaysBuild( "addMongodReqNoJsTargets" )

testEnv.Alias( "smokeAllNoJs", [ "smoke", "mongosTest", "addMongodReqNoJsTargets" ] )
testEnv.AlwaysBuild( "smokeAllNoJs" )

import atexit
atexit.register( stopMongodForTests )

def machine_info(extra_info=""):
    """Get a dict representing the "machine" section of a benchmark result.

    ie:
    {
        "os_name": "OS X",
        "os_version": "10.5",
        "processor": "2.4 GHz Intel Core 2 Duo",
        "memory": "3 GB 667 MHz DDR2 SDRAM",
        "extra_info": "Python 2.6"
    }

    Must have a settings.py file on sys.path that defines "processor" and "memory"
    variables.
    """
    sys.path.append("")
    import settings

    machine = {}
    (machine["os_name"], _, machine["os_version"], _, _) = os.uname()
    machine["processor"] = settings.processor
    machine["memory"] = settings.memory
    machine["extra_info"] = extra_info
    return machine

def post_data(data, machine_extra_info="", post_url="http://mongo-db.appspot.com/benchmark"):
    """Post a benchmark data point.

    data should be a Python dict that looks like:
        {
          "benchmark": {
            "project": "http://github.com/mongodb/mongo-python-driver",
            "name": "insert test",
            "description": "test inserting 10000 documents with the C extension enabled",
            "tags": ["insert", "python"]
          },
          "trial": {
            "server_hash": "4f5a8d52f47507a70b6c625dfb5dbfc87ba5656a",
            "client_hash": "8bf2ad3d397cbde745fd92ad41c5b13976fac2b5",
            "result": 67.5,
            "extra_info": "some logs or something"
          }
        }
    """
    try:
        import json
    except:
        import simplejson as json # needed for python < 2.6

    data["machine"] = machine_info(machine_extra_info)
    print( data )
    urllib2.urlopen(post_url, urllib.urlencode({"payload": json.dumps(data)}))

def recordPerformance( env, target, source ):
    global perftest
    import subprocess, re
    p = subprocess.Popen( [ perftest[0].abspath ], stdout=subprocess.PIPE )
    b = p.communicate()[ 0 ]
    print( "perftest results:" );
    print( b );
    if p.returncode != 0:
        return True
    entries = re.findall( "{.*?}", b )
    import sys
    for e in entries:
        matches = re.match( "{'(.*?)': (.*?)}", e )
        name = matches.group( 1 )
        val = float( matches.group( 2 ) )
        sub = { "benchmark": { "project": "http://github.com/mongodb/mongo", "description": "" }, "trial": {} }
        sub[ "benchmark" ][ "name" ] = name
        sub[ "benchmark" ][ "tags" ] = [ "c++", re.match( "(.*)__", name ).group( 1 ) ]
        sub[ "trial" ][ "server_hash" ] = getGitVersion()
        sub[ "trial" ][ "client_hash" ] = ""
        sub[ "trial" ][ "result" ] = val
        try:
            post_data(sub)
        except:
            print( "exception posting perf results" )
            print( sys.exc_info() )
    return False

addSmoketest( "recordPerf", [ "perftest" ] , [ recordPerformance ] )

#  ----  INSTALL -------

if distBuild:
    from datetime import date
    today = date.today()
    installDir = "mongodb-" + platform + "-" + processor + "-";
    if distName is None:
        installDir += today.strftime( "%Y-%m-%d" )
    else:
        installDir += distName
    print "going to make dist: " + installDir

# binaries

allBinaries = []

def installBinary( e , name ):
    global allBinaries
    if windows:
        name += ".exe"
    env.Install( installDir + "/bin" , name )
    allBinaries += [ name ]

installBinary( env , "mongodump" )
installBinary( env , "mongorestore" )

installBinary( env , "mongoexport" )
installBinary( env , "mongoimportjson" )

installBinary( env , "mongofiles" )

installBinary( env , "mongod" )
installBinary( env , "mongos" )

if not noshell:
    installBinary( env , "mongo" )

env.Alias( "all" , allBinaries )


# NOTE: In some cases scons gets confused between installation targets and build
# dependencies.  Here, we use InstallAs instead of Install to prevent such confusion
# on a case-by-case basis.

#headers
for id in [ "", "util/", "db/" , "client/" ]:
    env.Install( installDir + "/include/mongo/" + id , Glob( id + "*.h" ) )

#lib
env.Install( installDir + "/" + nixLibPrefix, clientLibName )
if usejvm:
    env.Install( installDir + "/" + nixLibPrefix + "/mongo/jars" , Glob( "jars/*" ) )

#textfiles
if distBuild or release:
    #don't want to install these /usr/local/ for example
    env.Install( installDir , "distsrc/README" )
    env.Install( installDir , "distsrc/THIRD-PARTY-NOTICES" )
    env.Install( installDir , "distsrc/GNU-AGPL-3.0" )

#final alias
env.Alias( "install" , installDir )

#  ---- CONVENIENCE ----

def tabs( env, target, source ):
    from subprocess import Popen, PIPE
    from re import search, match
    diff = Popen( [ "git", "diff", "-U0", "origin", "master" ], stdout=PIPE ).communicate()[ 0 ]
    sourceFile = False
    for line in diff.split( "\n" ):
        if match( "diff --git", line ):
            sourceFile = not not search( "\.(h|hpp|c|cpp)\s*$", line )
        if sourceFile and match( "\+ *\t", line ):
            return True
    return False
env.Alias( "checkSource", [], [ tabs ] )
env.AlwaysBuild( "checkSource" )

def gitPush( env, target, source ):
    import subprocess
    return subprocess.call( [ "git", "push" ] )
env.Alias( "push", [ ".", "smoke", "checkSource" ], gitPush )
env.AlwaysBuild( "push" )


# ---- deploying ---

def s3push( localName , remoteName=None , remotePrefix=None , fixName=True , platformDir=True ):

    if remotePrefix is None:
        if distName is None:
            remotePrefix = "-latest"
        else:
            remotePrefix = "-" + distName

    sys.path.append( "." )

    import simples3
    import settings

    s = simples3.S3Bucket( settings.bucket , settings.id , settings.key )

    if remoteName is None:
        remoteName = localName

    if fixName:
        (root,dot,suffix) = localName.rpartition( "." )
        name = remoteName + "-" + platform + "-" + processor + remotePrefix
        if dot == "." :
            name += "." + suffix
        name = name.lower()
    else:
        name = remoteName

    if platformDir:
        name = platform + "/" + name

    print( "uploading " + localName + " to http://s3.amazonaws.com/" + s.name + "/" + name )
    s.put( name  , open( localName , "rb" ).read() , acl="public-read" );
    print( "  done uploading!" )

def s3shellpush( env , target , source ):
    s3push( "mongo" , "mongo-shell" )

env.Alias( "s3shell" , [ "mongo" ] , [ s3shellpush ] )
env.AlwaysBuild( "s3shell" )

def s3dist( env , target , source ):
    s3push( distFile , "mongodb" )

env.Append( TARFLAGS=" -z " )
if windows:
    distFile = installDir + ".zip"
    env.Zip( distFile , installDir )
else:
    distFile = installDir + ".tgz"
    env.Tar( distFile , installDir )

env.Alias( "dist" , distFile )
env.Alias( "s3dist" , [ "install"  , distFile ] , [ s3dist ] )
env.AlwaysBuild( "s3dist" )

def clean_old_dist_builds(env, target, source):
    prefix = "mongodb-%s-%s" % (platform, processor)
    filenames = sorted(os.listdir("."))
    filenames = [x for x in filenames if x.startswith(prefix)]
    to_keep = [x for x in filenames if x.endswith(".tgz") or x.endswith(".zip")][-2:]
    for filename in [x for x in filenames if x not in to_keep]:
        print "removing %s" % filename
        try:
            shutil.rmtree(filename)
        except:
            os.remove(filename)

env.Alias("dist_clean", [], [clean_old_dist_builds])
env.AlwaysBuild("dist_clean")
