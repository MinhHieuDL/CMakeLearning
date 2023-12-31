cmake_minimum_required(VERSION 3.25)

project(function_And_macro)

# 8.2 - 8.3 Argument Handling Essentials

# WARNING: This macro is misleading
# Because macros treat their arguments as string substitutions rather than as variables, if they use ARGN in a
# place where a variable name is expected, the variable it will refer to will be in the scope from
# which the macro is called, not the ARGN from the macro’s own arguments. 
# The below example will have output is: "Argument: 3"
macro(dangerous)
    # Which ARGN?
    foreach(arg IN LISTS ARGN)
    # To fix the issue use: foreach(arg IN ITEMS ${ARGN})
        message("Argument: ${arg}")
    endforeach()
endmacro()

function(func)
    dangerous(1 2)
endfunction()

func(3)

function(func1)
    # Define the supported set of keywords
    set(prefix ARG)
    set(noValues ENABLE_NET COOL_STUFF)
    set(singleValues TARGET)
    set(multiValues SOURCES IMAGES)

    # Process the arguments passed in
    cmake_parse_arguments(
        ${prefix}
        "${noValues}" "${singleValues}" "${multiValues}"
        ${ARGN}
    )

    # Log details for each supported keyword
    message("Option summary:")

    foreach(arg IN LISTS noValues)
        if(${prefix}_${arg})
            message("   ${arg} enabled")
        else()
            message("   ${arg} disabled")
        endif()
    endforeach()

    foreach(arg IN LISTS singleValues multiValues)
        # Single argument values will print as a string
        # Multiple argument values will print as a list
        message("   ${arg} = ${${prefix}_${arg}}")
    endforeach()

    message("   unparsed list:")
    foreach(arg IN LISTS ${prefix}_UNPARSED_ARGUMENTS)
        message("       ${arg}")
    endforeach()
    
endfunction()

func1(SOURCES foo.cpp bar.cpp
      TARGET myApp
      ENABLE_NET
)

func1(not_parsed nothing_to_do
      COOL_STUFF
      TARGET dummy
      IMAGES here.png there.png gone.png 
      )

function(demoArgs)
    set(prefix ARG)
    set(noValues "")
    set(singleValues SPECIAL)
    set(multiValues  EXTRAS)

    # Should use the first one, since:
    # the advantage of the PARSE_ARGV form is that if any unparsed
    # arguments are themselves a list, their embedded semicolons will be escaped. This preserves the
    # original structure of the arguments, unlike the other form of the command which doesn’t.
    cmake_parse_arguments(
        PARSE_ARGV 0
        ${prefix}
        "${noValues}" "${singleValues}" "${multiValues}"
    )
    # cmake_parse_arguments(
    #     ${prefix}
    #     "${noValues}" "${singleValues}" "${multiValues}"
    #     ${ARGN}
    # )
    
    message("Left-over args: ${${prefix}_UNPARSED_ARGUMENTS}")
    foreach(arg IN LISTS ${prefix}_UNPARSED_ARGUMENTS)
        message(${arg})
    endforeach()
    
endfunction()


demoArgs(burger fries "cheese;tomato" SPECIAL secretSauce)

# Multi-level argument parsing:
function(libWithTest)
    # First level of arguments
    set(group LIB TEST)
    cmake_parse_arguments(GRP "" "" "${group}" ${ARGN})

    # Second level of arguments
    set(args SOURCES PRIVATE_LIBS PUBLIC_LIBS)
    cmake_parse_arguments(LIB "" "TARGET" "${args}" ${GRP_LIB})
    cmake_parse_arguments(TEST "" "TARGET" "${args}" ${GRP_TEST})
    
    add_library(${LIB_TARGET} ${LIB_SOURCES})
    target_link_libraries(${LIB_TARGET}
        PUBLIC  ${LIB_PUBLIC_LIBS}
        PRIVATE ${LIB_PRIVATE_LIBS}
    )

    add_executable(${TEST_TARGET} ${TEST_SOURCES})
    target_link_libraries(${TEST_TARGET}
        PUBLIC  ${TEST_PUBLIC_LIBS}
        PRIVATE ${TEST_PRIVATE_LIBS}
    )
endfunction()

# libWithTest(
#     LIB
#         TARGET Algo
#         SOURCES algo.cpp algo.h
#         PUBLIC_LIBS SomeMathHelpers
#     TEST
#         TARGET AlgoTest
#         SOURCES algoTest.cpp
#         PRIVATE_LIBS gtest_main
# )

# 8.4 Returning Values
function(doSomething outVar)
    set(${outVar} 42)
    return(PROPAGATE ${outVar})
endfunction()

set(result 24)
doSomething(result)
# Here, a variable named result now should holds the value 42
message("result: ${result}")

# block() statements within the function do not prevent propagation
# to the function’s caller

function(doChangeVar outVar)
    set(${outVar} 42)
    block()
        set(${outVar} 27)
        return(PROPAGATE ${outVar})
    endblock()
endfunction()

doChangeVar(result)
# Here, a variable named result now should holds the value 27
message("result: ${result}")

# With CMake 3.24 and earlier, functions do not support returning a value directly.
# Use set() and unset() with PARENT_SCOPE keyword to modify the variable in caller scope
function(changeVarOfCallerScope outVar1 outVar2)
    set(${outVar1} "First Var 1" PARENT_SCOPE)
    set(${outVar2} "First Var 2" PARENT_SCOPE)
endfunction()

changeVarOfCallerScope(myVar otherVar)

message("myVar: ${myVar}")
message("otherVar: ${otherVar}")

# The function changeVarOfCallerScope can be implemented using macro like below:
macro(changeVarOfCallerScopeMacr outVar1 outVar2)
    set(${outVar1} "First Var 1")
    set(${outVar2} "First Var 2")
endmacro()

# Calling cmake code using cmake_language cmd with CALL sub-command
# The following example demonstrates how a generic wrapper can be defined around a set of
# functions that include a version number in their name
function(qt_generate_moc)
    set(cmd qt${QT_DEFAULT_MAJOR_VERSION}_generate_moc)

    cmake_language(CALL ${cmd} ${ARGV})
endfunction()
# The above example assumes the QT_DEFAULT_MAJOR_VERSION variable has been set previously. As future
# Qt major versions are released, the above would continue to work as long as the appropriate
# versioned command was still provided. The alternative would be to implement an ever-expanding
# set of if() tests for each version individually

# Calling cmake code using cmake_language cmd with EVAL sub-command

set(myProjTraceCall [=[
    message("Called ${CMAKE_CURRENT_FUNCTION}")
    set(__x 0)
    while(__x LESS ${ARGC})
        message("   ARGV${__x} = ${ARGV${__x}}")
        math(EXPR __x "${__x} + 1")
    endwhile()
    unset(__x)
]=])

function(testTrace)
    cmake_language(EVAL CODE "${myProjTraceCall}")
endfunction()

testTrace(one two tree)