import os.path, copy, sys

# Inspired by the versioning scheme followed by Qt, it seems sensible enough. There are three components: major, minor
# and micro. Major changes with each subtraction from the API (backward-incompatible, i.e. V19 vs. V18), minor changes
# with each addition to the API (backward-compatible), micro changes with each revision of the source code.
ApiVer = "2.0.0"

def checkSymbol(conf, header, library=None, symbol=None, autoAdd=True, critical=False, pkgName=None):
    """ Check for symbol in library, optionally look only for header.
    @param conf: Configure instance.
    @param header: The header file where the symbol is declared.
    @param library: The library in which the symbol exists, if None it is taken to be the standard C library.
    @param symbol: The symbol to look for, if None only the header will be looked up.
    @param autoAdd: Automatically link with this library if check is positive.
    @param critical: Raise on error?
    @param pkgName: Optional name of pkg-config entry for library, to determine build parameters.
    @return: True/False
    """
    origEnv = conf.env.Copy() # Copy unmodified environment so we can restore it upon error
    env = conf.env
    if library is None:
        library = "c"   # Standard library
        autoAdd = False

    if pkgName is not None:
        origLibs = copy.copy(env.get("LIBS", None))

        try: env.ParseConfig("pkg-config --silence-errors %s --cflags --libs" % pkgName)
        except: pass
        else:
            # I see no other way of checking that the parsing succeeded, if it did add no more linking parameters
            if env.get("LIBS", None) != origLibs:
                autoAdd = False

    try:
        if not conf.CheckCHeader(header, include_quotes="<>"):
            raise ConfigurationError("missing header %s" % header)
        if symbol is not None and not conf.CheckLib(library, symbol, language="C", autoadd=autoAdd):
            raise ConfigurationError("missing symbol %s in library %s" % (symbol, library))
    except ConfigurationError:
        conf.env = origEnv
        if not critical:
            return False
        raise

    return True

import SCons.Errors

# Import common variables

try:
    Import("Platform", "Posix", "ConfigurationError")
except SCons.Errors.UserError:
    # The common objects must be exported first
    SConscript(os.path.join("scons", "SConscript_common"))
    Import("Platform", "Posix", "ConfigurationError")

Import("env")

# This will be manipulated
env = env.Copy()

env.Append(CPPPATH="pa_common")

# Store all signatures in one file, otherwise .sconsign files will get installed along with our own files
env.SConsignFile(os.path.join("scons", ".sconsign"))

# We operate with a set of needed libraries and optional libraries, the latter stemming from host API implementations.
# For libraries of both types we record a set of values that is used to look for the library in question, during
# configuration. If the corresponding library for a host API implementation isn't found, the implementation is left out.
neededLibs = []
optionalImpls = {}
if Platform in Posix:
    neededLibs += [("pthread", "pthread.h", "pthread_create"), ("m", "math.h", "sin")]
    if env["useALSA"]:
        optionalImpls["ALSA"] = ("asound", "alsa/asoundlib.h", "snd_pcm_open")
    if env["useJACK"]:
        optionalImpls["JACK"] = ("jack", "jack/jack.h", "jack_client_new")
    if env["useOSS"]:
        # TODO: It looks like the prefix for soundcard.h depends on the platform
        optionalImpls["OSS"] = ("oss", "sys/soundcard.h", None)
else:
    raise ConfigurationError("unknown platform %s" % Platform)

if Platform == "darwin":
    env.Append(LINKFLAGS=["-framework CoreAudio", "-framework AudioToolBox"])
    env.Append(CPPDEFINES=["PA_USE_COREAUDIO"])
elif Platform == "cygwin":
    env.Append(LIBS=["winmm"])
elif Platform == "irix":
    neededLibs +=  [("audio", "dmedia/audio.h", "alOpenPort"), ("dmedia", "dmedia/dmedia.h", "dmGetUST")]
    env.Append(CPPDEFINES=["PA_USE_SGI"])

def CheckCTypeSize(context, tp):
    """ Check size of C type.
    @param context: A configuration context.
    @param tp: The type to check.
    @return: Size of type, in bytes.
    """
    context.Message("Checking the size of C type %s..." % tp)
    ret = context.TryRun("""
#include <stdio.h>

int main() {
    printf("%%d", sizeof(%s));
    return 0;
}
""" % tp, ".c")
    if not ret[0]:
        print "Hihi"
        context.Result(" Couldn't obtain size of type %s!" % tp)
        return None

    assert ret[1]
    sz = int(ret[1])
    context.Result("%d" % sz)
    return sz

if sys.byteorder == "little":
    env.Append(CPPDEFINES=["PA_LITTLE_ENDIAN"])
elif sys.byteorder == "big":
    env.Append(CPPDEFINES=["PA_BIG_ENDIAN"])
else:
    raise ConfigurationError("unknown byte order: %s" % sys.byteorder)
if env["enableDebugOutput"]:
    env.Append(CPPDEFINES=["PA_ENABLE_DEBUG_OUTPUT"])

# Start configuration

# Use an absolute path for conf_dir, otherwise it gets created both relative to current directory and build directory
conf = env.Configure(log_file=os.path.join("scons", "sconf.log"), custom_tests={"CheckCTypeSize": CheckCTypeSize},
        conf_dir=os.path.join(os.getcwd(), "scons", ".sconf_temp"))
assert os.path.exists(os.path.join("scons", ".sconf_temp"))
env.Append(CPPDEFINES=["SIZEOF_SHORT=%d" % conf.CheckCTypeSize("short")])
env.Append(CPPDEFINES=["SIZEOF_INT=%d" % conf.CheckCTypeSize("int")])
env.Append(CPPDEFINES=["SIZEOF_LONG=%d" % conf.CheckCTypeSize("long")])
if checkSymbol(conf, "time.h", "rt", "clock_gettime"):
    env.Append(CPPDEFINES=["HAVE_CLOCK_GETTIME"])
if checkSymbol(conf, "time.h", symbol="nanosleep"):
    env.Append(CPPDEFINES=["HAVE_NANOSLEEP"])

# Look for needed libraries and link with them
for lib, hdr, sym in neededLibs:
    checkSymbol(conf, hdr, lib, sym, critical=True)
# Look for host API libraries, if a library isn't found disable corresponding host API implementation.
for name, val in optionalImpls.items():
    lib, hdr, sym = val
    if checkSymbol(conf, hdr, lib, sym, critical=False, pkgName=name.lower()):
        env.Append(CPPDEFINES=["PA_USE_%s=1" % name.upper()])
    else:
        del optionalImpls[name]

# Configuration finished
env = conf.Finish()

# PA infrastructure
CommonSources = [os.path.join("pa_common", f) for f in "pa_allocation.c pa_converters.c pa_cpuload.c pa_dither.c pa_front.c \
        pa_process.c pa_skeleton.c pa_stream.c pa_trace.c".split()]

# Host API implementations
ImplSources = []
if Platform in Posix:
    ThreadCFlags = "-pthread"
    env.AppendUnique(LINKFLAGS=[ThreadCFlags])
    BaseCFlags = "-Wall -pedantic -pipe %s" % ThreadCFlags
    DebugCFlags = "-g"
    OptCFlags  = "-O2"
    ImplSources += [os.path.join("pa_unix", f) for f in "pa_unix_hostapis.c pa_unix_util.c".split()]

if "ALSA" in optionalImpls:
    ImplSources.append(os.path.join("pa_linux_alsa", "pa_linux_alsa.c"))
if "JACK" in optionalImpls:
    ImplSources.append(os.path.join("pa_jack", "pa_jack.c"))
if "OSS" in optionalImpls:
    ImplSources.append(os.path.join("pa_unix_oss", "pa_unix_oss.c"))

env["CCFLAGS"] = BaseCFlags
if env["enableDebug"]:
    env["CCFLAGS"] += " " + DebugCFlags
if env["enableOptimize"]:
    env["CCFLAGS"] += " " + OptCFlags
if not env["enableAsserts"]:
    env.Append(CPPDEFINES=["-DNDEBUG"])
if env["customCFlags"]:
    env["CCFLAGS"] += " " + env["customCFlags"]
if env["customLinkFlags"]:
    env["LINKFLAGS"] += " " + env["customLinkFlags"]

sources = CommonSources + ImplSources

if env["enableShared"]:
    sharedLibEnv = env.Copy()
    if Platform in Posix:
        # Add soname to library, this is so a reference is made to the versioned library in programs linking against libportaudio.so
        sharedLibEnv.AppendUnique(SHLINKFLAGS="-Wl,-soname=libportaudio.so.%d" % int(ApiVer.split(".")[0]))
    sharedLib = sharedLibEnv.SharedLibrary(target="portaudio", source=sources)
else:
    sharedLib = None
staticLib = env.StaticLibrary(target="portaudio", source=sources)

if Platform in Posix:
    prefix = env["prefix"]
    includeDir = os.path.join(prefix, "include")
    env.Alias("install", env.Install(includeDir, os.path.join("pa_common", "portaudio.h")))
    libDir = os.path.join(prefix, "lib")

    if env["enableStatic"]:
        env.Alias("install", env.Install(libDir, staticLib))

    def symlink(env, target, source):
        trgt = str(target[0])
        src = str(source[0])

        if os.path.lexists(trgt):
            os.remove(trgt)
        os.symlink(os.path.basename(src), trgt)

    if env["enableShared"]:
        major, minor, micro = [int(c) for c in ApiVer.split(".")]
        
        soFile = "%s.%s" % (sharedLib[0], ApiVer)
        env.Alias("install", env.InstallAs(target=os.path.join(libDir, soFile), source=sharedLib))
        # Install symlinks
        symTrgt = os.path.join(libDir, soFile)
        env.Alias("install", env.Command(os.path.join(libDir, "libportaudio.so.%d.%d" % (major, minor)),
            symTrgt, symlink))
        symTrgt = symTrgt.rsplit(".", 1)[0]
        env.Alias("install", env.Command(os.path.join(libDir, "libportaudio.so.%d" % major), symTrgt, symlink))
        symTrgt = symTrgt.rsplit(".", 1)[0]
        env.Alias("install", env.Command(os.path.join(libDir, "libportaudio.so"), symTrgt, symlink))

    # pkg-config

    def installPkgconfig(env, target, source):
        tgt = str(target[0])
        src = str(source[0])
        f = open(src)
        try: txt = f.read()
        finally: f.close()
        txt = txt.replace("@prefix@", prefix)
        txt = txt.replace("@libdir@", libDir)
        txt = txt.replace("@includedir@", includeDir)
        txt = txt.replace("@LIBS@", " ".join(["-l%s" % l for l in env["LIBS"]]))
        txt = txt.replace("@THREAD_CFLAGS@", ThreadCFlags)

        f = open(tgt, "w")
        try: f.write(txt)
        finally: f.close()

    pkgconfigTgt = "portaudio-%d.0.pc" % int(ApiVer.split(".", 1)[0])
    env.Alias("install", env.Command(os.path.join(libDir, "pkgconfig", pkgconfigTgt),
        pkgconfigTgt + ".in", installPkgconfig))

testNames = ["patest_sine", "paqa_devs", "paqa_errs", "patest1", "patest_buffer", "patest_callbackstop", "patest_clip", \
        "patest_dither", "patest_hang", "patest_in_overflow", "patest_latency", "patest_leftright", "patest_longsine", \
        "patest_many", "patest_maxsines", "patest_multi_sine", "patest_out_underflow", "patest_pink", "patest_prime", \
        "patest_read_record", "patest_record", "patest_ringmix", "patest_saw", "patest_sine8", "patest_sine", \
        "patest_sine_time", "patest_start_stop", "patest_stop", "patest_sync", \
        "patest_toomanysines", "patest_underflow", "patest_wire", "patest_write_sine", "pa_devs", "pa_fuzz", "pa_minlat"]

# The test directory ("bin") should be in the top-level PA directory, the calling script should ensure that SCons doesn't
# switch current directory by calling SConscriptChdir(False). Specifying an absolute directory for each target means it won't
# be relative to the build directory
tests = [env.Program(target=os.path.join(os.getcwd(), "bin", name), source=[os.path.join("pa_tests", name + ".c"),
        staticLib]) for name in testNames]

Return("sources", "sharedLib", "staticLib", "tests", "env")