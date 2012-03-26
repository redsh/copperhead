from __future__ import print_function
import os
import re
import subprocess
from distutils.errors import CompileError
import operator

# What are we doing?
# Figure out whether we're copying Python files
# Or building C++ extensions, or both
python_build = False
ext_build = False
if 'build_py' in COMMAND_LINE_TARGETS:
    python_build = True
if 'build_ext' in COMMAND_LINE_TARGETS:
    ext_build = True
if not python_build and not ext_build:
    python_build = True
    ext_build = True


# try to import an environment first
try:
    Import('env')
except:
    exec open(os.path.join('config', "build-env.py"))
    env = Environment()

# Check availability of a program, with given version.
def CheckVersion(context, cmd, exp, required, extra_error=''):
    context.Message("Checking {cmd} version... ".format(cmd=cmd.split()[0]))

    try:
        log = context.sconf.logstream if context.sconf.logstream else file('/dev/null','w')
        vsout = subprocess.check_output([cmd], shell=True, stderr=log)
    except subprocess.CalledProcessError:
        context.Result('%s was not found %s' % (cmd.split()[0], extra_error) )
        return False
    match = exp.search(vsout)
    if not match:
        context.Result('%s returned unexpected output' % (cmd, extra_error) )
        return False
    version = match.group(1)
    exploded_version = version.split('.')
    if not all(map(operator.le, required, exploded_version)):
        context.Result("%s returned version %s, but we need version %s or better." % (cmd, version, '.'.join(required), extra_error) )
        return False
    context.Result(version)
    return True

thrust_version_check_file = """
#define THRUST_DEVICE_SYSTEM THRUST_DEVICE_SYSTEM_OMP
#include <thrust/version.h>
#include <iostream>
int main() {
  std::cout << THRUST_MAJOR_VERSION << std::endl;
  std::cout << THRUST_MINOR_VERSION << std::endl;
  std::cout << THRUST_SUBMINOR_VERSION << std::endl;
  return 0;
}
"""

def CheckThrustVersion(context, required_version):
    context.Message("Checking Thrust version...")
    int_required_version = [int(x) for x in required_version]
    result = context.TryRun(thrust_version_check_file, ".cpp")[1]
    returned_version = result.splitlines(False)
    version = '.'.join(returned_version)
    context.Result(version)

    int_returned_version = [int(x) for x in returned_version]
    return all(map(operator.le, int_required_version, int_returned_version))

def autoconf():
    conf=Configure(env, custom_tests = {'CheckVersion':CheckVersion,
                                        'CheckThrustVersion':CheckThrustVersion})
    siteconf = {}

    #Ensure we have g++ >= 4.5
    gpp_re = re.compile(r'g\+\+ \(.*\) ([\d\.]+)')
    conf.CheckVersion('g++ --version', gpp_re, (4,5))

    # Check to see if we have nvcc >= 4.1
    nv_re = re.compile(r'release ([\d\.]+)')
    cuda_support=conf.CheckVersion('nvcc --version',
                                   nv_re,
                                   (4,1),
                                   extra_error="nvcc was not found. No CUDA support will be included.")
    Export('cuda_support')
    # Check we have numpy
    try:
        import numpy
        np_path, junk = os.path.split(numpy.__file__)
        np_inc_path = os.path.join(np_path, 'core', 'include')
        siteconf['NP_INC_PATH'] = np_inc_path

    except ImportError:
        raise CompileError("numpy must be installed before building copperhead")

    # Check to see if the user has written down siteconf stuff
    if os.path.exists("siteconf.py"):
        glb = {}
        execfile("siteconf.py", glb, siteconf)
    else:
        print("""
*************** siteconf.py not found ***************
We will try building anyway, but may not succeed.
Read the README for more details.
""")

        siteconf['BOOST_INC_DIR'] = None
        siteconf['BOOST_LIB_DIR'] = None
        siteconf['BOOST_PYTHON_LIBNAME'] = None
        siteconf['THRUST_PATH'] = None

        f = open("siteconf.py", 'w')
        print("""#! /usr/bin/env python
#
# Configuration file.
# Use Python syntax, e.g.:
# VARIABLE = "value"
# 
# The following information can be recorded:
#
# BOOST_INC_DIR : Directory where the Boost include files are found.
#
# BOOST_LIB_DIR : Directory where Boost shared libraries are found.
#
# BOOST_PYTHON_LIBNAME : Name of Boost::Python shared library.
#   NOTE: Boost::Python must be compiled using the same compiler
#   that was used to build your Python.  Strange errors will
#   ensue if this is not true.
# 
# THRUST_PATH : Directory where Thrust include files are found.
#
# NP_INC_PATH: Directory where Numpy include files are found.
#
""", file=f)


        for k, v in siteconf.items():
            if v:
                v = '"' + str(v) + '"'
            print('%s = %s' % (k, v), file=f)

        f.close()

    Export('siteconf')

    if siteconf['BOOST_INC_DIR']:
        env.Append(CPPPATH=siteconf['BOOST_INC_DIR'])
    if siteconf['BOOST_LIB_DIR']:
        env.Append(LIBPATH=siteconf['BOOST_LIB_DIR'])
    if siteconf['THRUST_PATH']:
        #Must prepend because build-env.py might have found an old system Thrust
        env.Prepend(CPPPATH=siteconf['THRUST_PATH'])

    # Check we have boost::python
    from distutils.sysconfig import get_python_lib, get_python_version
    env.Append(LIBPATH=os.path.join(get_python_lib(0,1),"config"))


    if not conf.CheckLib('python'+get_python_version(), language="C++"):
        print("You need the python development library to compile this program")
        Exit(1)

    if not siteconf['BOOST_PYTHON_LIBNAME']:
        siteconf['BOOST_PYTHON_LIBNAME'] = 'boost_python'

    if not conf.CheckLib(siteconf['BOOST_PYTHON_LIBNAME'], language="C++"):
        print("You need the Boost::Python library to compile this program")
        print("Consider installing it, or changing BOOST_PYTHON_LIBNAME in siteconf.py")
        Exit(1)

    #Check we have a Thrust installation
    if not conf.CheckCXXHeader('thrust/host_vector.h'):
        print("You need Thrust to compile this program")
        print("Consider installing it, or changing THRUST_PATH in siteconf.py")
        Exit(1)

    #Ensure Thrust Version > 1.6
    if not conf.CheckThrustVersion((1,6)):
        print("You need Thrust version 1.6 or greater")
        print("Change THRUST_PATH in siteconf.py to point to your Thrust installation.")
        Exit(1)
        
    # MacOS Support
    if env['PLATFORM'] == 'darwin':
        env.Append(SHLINKFLAGS = '-undefined dynamic_lookup')


    #Parallelize the build maximally
    import multiprocessing
    n_jobs = multiprocessing.cpu_count()
    SetOption('num_jobs', n_jobs)

# Configuration is expensive, only do it if we need to
if ext_build:
    autoconf()
    
Export('env')

#Build backend
build_ext_targets = []

libcopperhead = SConscript(os.path.join('backend', 'src', 'SConscript'),
                           variant_dir=os.path.join('backend','build'),
                           duplicate=0)

Export('libcopperhead')


build_ext_targets.append(env.Install(os.path.join('stage', 'copperhead', 'runtime'), libcopperhead))

extensions = SConscript(os.path.join('src', 'SConscript'),
                            variant_dir='stage',
                            duplicate=0)
build_ext_targets.extend(extensions)

def recursive_glob(pattern, dir=os.curdir):
    files = Glob(dir+'/'+pattern)
    if files:
        files += recursive_glob("*/"+pattern,dir)
    return files

def explode_path(path):
    head, tail = os.path.split(path)
    return explode_path(head) + [tail] \
        if head and head != path \
        else [head or tail]


python_files = recursive_glob('*.py','copperhead')
build_py_targets = []

for x in python_files:
    head, tail = os.path.split(str(x))
    build_py_targets.append(env.Install(os.path.join('stage', head), x))
    
    library_files = recursive_glob('*.h*', os.path.join(
        'backend', 'inc', 'prelude'))

    for x in library_files:
        exploded_path = explode_path(str(x))[:-1]
        exploded_path[0] = os.path.join('stage', 'copperhead')
        install_path = os.path.join(*exploded_path)
        build_py_targets.append(env.Install(install_path, x))

    frontend_library_files = recursive_glob('*.hpp', os.path.join(
        'src', 'copperhead', 'runtime'))

    backend_library_files = [os.path.join('backend','inc', x+'.hpp') for x in \
                             ['type', 'monotype', 'polytype', 'ctype']]

    for x in frontend_library_files + backend_library_files:
        install_path = os.path.join('stage','copperhead', 'inc', 'prelude', 'runtime')
        build_py_targets.append(env.Install(install_path, x))

    siteconf_file = 'siteconf.py'
    build_py_targets.append(env.Install(os.path.join('stage', 'copperhead', 'runtime'),
                                        siteconf_file))

if 'build_py' in COMMAND_LINE_TARGETS:
    env.Alias('build_py', build_py_targets)
    env.Default(build_py_targets)
if 'build_ext' in COMMAND_LINE_TARGETS:
    env.Alias('build_ext', build_ext_targets)
    env.Default(build_ext_targets)
