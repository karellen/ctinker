# CTinker - C/C++ Project Introspection and Augmentation Tool

**C Tinker**, pronounced _see-tinker_ (or humorously "stinker", as suggested by 
[Chuck Ocheret](https://github.com/ocheret)) allows you to get in the middle of the build process of a 
make/Ninja-style project and augment the compilation and linking as well as extract and redirect artifacts using 
policies you can't implement otherwise even with LDFLAGS/CFLAGS magic.

[![Gitter](https://img.shields.io/gitter/room/karellen/lobby?logo=gitter)](https://gitter.im/karellen/lobby)

[![CTinker Version](https://img.shields.io/pypi/v/ctinker?logo=pypi)](https://pypi.org/project/ctinker/)
[![CTinker Python Versions](https://img.shields.io/pypi/pyversions/ctinker?logo=pypi)](https://pypi.org/project/ctinker/)
[![CTinker Downloads Per Day](https://img.shields.io/pypi/dd/ctinker?logo=pypi)](https://pypi.org/project/ctinker/)
[![CTinker Downloads Per Week](https://img.shields.io/pypi/dw/ctinker?logo=pypi)](https://pypi.org/project/ctinker/)
[![CTinker Downloads Per Month](https://img.shields.io/pypi/dm/ctinker?logo=pypi)](https://pypi.org/project/ctinker/)

## Problem

More formally the problem **CTinker** solves can be stated as follows: 

> I need to get in the middle of a build process of a project I can know intimately but do not control
> and that I have no intention of maintaining a fork/patches for, or for which I need to obtain runtime 
> dynamic control of the build process.

## Solution

### Overview

`CTinker` is capable of getting in the middle of virtually any build process by: 
1. Starting in the `supervisor` mode.
1. Creating a temporary directory full of toolkit-specific (e.g. for LLVM Clang it's `clang`, `ar` etc) 
symlinks referring back to `CTinker` executable. 
1. Setting up environ and a local socket to communicate with the `workers`.
1. Invoking the build process as specified by the user.
1. Being invoked for each tool invocation in a `worker` mode (based on environmental variables),
 communicating with the `supervisor`, sending command-line arguments to the `supervisor` process and then
 invoking the tool itself.
1. If specified, invoking `scripting` handlers before and after the build as a whole (in the `supervisor`) 
and before and after each intercepted tool invocation (in the `worker`). 
 
 As a further illustration, if the original process invocation chain for a sample build is as follows:
 
> make => clang => lld, => make => clang, => clang => lld
 
 then the same build instrumented with CTinker will produce the following process invocation chain:

> ctinker => make => ctinker-clang => clang => ctinker-lld => lld, => make => ctinker-clang => clang, 
> => ctinker-clang => clang => ctinker-lld => lld

### Scripting

Scripting is the most powerful part of `CTinker` that provides an ability to really change how build functions
at runtime. It is implemented via a visitor pattern, invoking functions specified in the user-supplied script:

```python

def ctinker_start(env: Dict[str, str], work_dir: Path):
    """Invoked by CTinker `supervisor` prior to the main build process
    
    Changes to the `env` dictionary propagate to the main build process.
    """
    pass

def ctinker_finish(env: Dict[str, str], work_dir: Path, tool_calls: List[Tuple[Any]], return_code: int):
    """Invoked by CTinker `supervisor` after the main build process exits

    `tool_calls` is a `list` of `tuple`s of `(tool, tool_args, return_code, cwd, script_result)`, where `script_result`
    is the value returned by `ctinker_after_tool`.
    """
    pass


def ctinker_before_tool(env: Dict[str, str], tool: str, tool_args: List[str], work_dir: Path, cwd: Path):
    """Invoked by CTinker `worker` prior to the tool process
    
    Changes to the `env` dictionary propagate to the tool process.
    Changes to the `tool_args` list propagate to the tool process.
    """
    pass

def ctinker_after_tool(env: Dict[str, str], tool: str, tool_args: List[str], work_dir: Path, cwd: Path, 
                       return_code: int) -> Any:
    """Invoked by CTinker `worker` after the tool process exits
    
    Returned value, **if truthy**, will be stored and will appear 
    as the last entry in the `tool_calls` passed to `ctinker_finish`
    """
    pass
```

It is guaranteed that `ctinker_start` - `ctinker_finish` and `ctinker_before_tool` - `ctinker_after_tool` pairs will 
be executed in the same `supervisor` and `worker` processes _respectively_ and therefore you can pass values between 
the start/finish and before/after functions (for example by a global or within the same instance of an object).

## Help

```bash
$ ctinker --help
usage: ctinker [-h] [-s SCRIPT] [-o OUT] [-f {text,pickle}] [-p PATHS]
               [-w WORK_DIR] -t {clang} [-l [{clang,__default}]]
               [-L LINKER_TOOL_OVERRIDE]
               [--linker-flags-name LINKER_FLAGS_NAME]
               ...

CTinker project introspection and augmentation tool

positional arguments:
  command               build command to execute

optional arguments:
  -h, --help            show this help message and exit
  -s SCRIPT, --script SCRIPT
                        a Python script containing visitor-style hooks
                        (default: None)
  -o OUT, --out OUT     a path to a file where all tools, their arguments and
                        exit codes will be recorded (default: None)
  -f {text,pickle}, --format {text,pickle}
                        the format of the output file (default: text)
  -p PATHS, --path PATHS
                        prepend a leading PATH entry to be inherited by the
                        invoked command (default: None)
  -w WORK_DIR, --work-dir WORK_DIR
                        sets a work directory to be something other than
                        current working directory (default: )
  -t {clang}, --toolkit {clang}
                        enable specific compilation interception modes
                        (default: None)
  -l [{clang,__default}], --linker-intercept [{clang,__default}]
                        intercept linker with --linker-flags-name env var
                        using the specified toolkit (default: None)
  -L LINKER_TOOL_OVERRIDE, --linker-tool-override LINKER_TOOL_OVERRIDE
                        specify linker tool name directly (may not work if no
                        toolkit provides it) (default: None)
  --linker-flags-name LINKER_FLAGS_NAME
                        specify linker environmental variable (default:
                        LDFLAGS)
```

## Example

TBW

## Troubleshooting

1. Printing to `sys.stdout` from the `worker` is dangerous as the stdout is often interpreted by the invoking tool
which can lead to a crash in the tool expecting certain data format. `print("debug!", file=sys.stderr)` is generally 
safe.