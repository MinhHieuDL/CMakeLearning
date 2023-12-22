cmake_minimum_required(VERSION 3.22)

project(function_And_macro)

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