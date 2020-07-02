cmake_minimum_required(VERSION 3.16)
project(IconManager2)

if(WIN32)
    set(CONVERT magick.exe convert)
    set(IDENTIFY magick.exe identify)
    set(INKSCAPE inkscape.com)
else()
    set(CONVERT convert)
    set(IDENTIFY identify)
    set(INKSCAPE env _INKSCAPE_GC=disable inkscape)
endif()

add_custom_target(RerunCmake ${CMAKE_COMMAND} ${CMAKE_SOURCE_DIR})

file(GLOB_RECURSE
        PNGs
        FOLLOW_SYMLINKS
        input/*.png
        )
#[[file(GLOB_RECURSE
        WEBPs
        FOLLOW_SYMLINKS
        input/*.webp
        )]]#
file(GLOB_RECURSE
        SVGs
        FOLLOW_SYMLINKS
        input/*.svg
        )

foreach(PNG IN LISTS PNGs)
    get_filename_component(Dir ${PNG} DIRECTORY)
    STRING(REGEX REPLACE "^${CMAKE_SOURCE_DIR}/input/" "" Dir ${Dir})
    get_filename_component(Name ${PNG} NAME_WE)
    list(APPEND IMGs ${Dir}/${Name})

    SET(IMGPath ${Dir}/${Name})
    STRING(REGEX REPLACE "/" "_" IMGTarget ${IMGPath})
    STRING(REGEX REPLACE " " "" IMGTarget ${IMGTarget})
    STRING(REGEX REPLACE "\\(" "" IMGTarget ${IMGTarget})
    STRING(REGEX REPLACE "\\)" "" IMGTarget ${IMGTarget})
    #message(${PNGTarget})

    #[[add_custom_command(
            OUTPUT ${PNGPath}
            COMMAND cp ${PNG} ${Dir}/${Name}.png
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )]]#
    #configure_file(${PNG} ${IMGPath}.png COPYONLY)
    add_custom_command(
            OUTPUT ${IMGPath}.png
            COMMAND ${CMAKE_COMMAND} -E make_directory ${Dir}
            COMMAND ${CMAKE_COMMAND} -E copy ${PNG} ${IMGPath}.png
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )

    add_custom_target(
            ${IMGTarget} ALL
            COMMAND :
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            DEPENDS ${IMGPath}.png
    )
endforeach()
#[[foreach(WEBP IN LISTS WEBPs)
    get_filename_component(Dir ${WEBP} DIRECTORY)
    STRING(REGEX REPLACE "^${CMAKE_SOURCE_DIR}/input/" "" Dir ${Dir})
    get_filename_component(Name ${WEBP} NAME_WE)
    list(APPEND IMGs ${Dir}/${Name})

    SET(IMGPath ${Dir}/${Name})
    STRING(REGEX REPLACE "/" "_" IMGTarget ${IMGPath})
    STRING(REGEX REPLACE " " "" IMGTarget ${IMGTarget})
    STRING(REGEX REPLACE "\\(" "" IMGTarget ${IMGTarget})
    STRING(REGEX REPLACE "\\)" "" IMGTarget ${IMGTarget})

    message(${IMGTarget})

    add_custom_command(
            OUTPUT ${PNGPath}
            COMMAND ${CONVERT} "${WEBP}" "PNG32:${IMGPath}.png"
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )

    add_custom_target(
            ${IMGTarget}.png ALL
            COMMAND echo ${IMGPath}
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            DEPENDS ${IMGPath}.png
    )
endforeach()]]#
foreach(SVG IN LISTS SVGs)
    get_filename_component(Dir ${SVG} DIRECTORY)
    STRING(REGEX REPLACE "^${CMAKE_SOURCE_DIR}/input/" "" Dir ${Dir})
    get_filename_component(Name ${SVG} NAME_WE)

    SET(IMGPath ${Dir}/${Name})
    STRING(REGEX REPLACE "/" "." IMGTarget ${IMGPath})
    STRING(REGEX REPLACE " " "_" IMGTarget ${IMGTarget})

    list(APPEND IMGs ${IMGPath})

    #message(${IMGTarget})

    add_custom_command(
            OUTPUT ${IMGPath}.png
            COMMAND ${CMAKE_COMMAND} -E make_directory ${Dir}
            COMMAND ${INKSCAPE} -o "${IMGPath}.png" -w 512 "${SVG}"
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )

    add_custom_target(
            ${IMGTarget} ALL
            COMMAND :
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            DEPENDS ${IMGPath}.png
    )
endforeach()

foreach(IMG IN LISTS IMGs)
    STRING(REGEX REPLACE "/" "." IMGTarget ${IMG})
    STRING(REGEX REPLACE " " "_" IMGTarget ${IMGTarget})

    #magick convert "$filedir\$filename$namemod.png" -trim "PNG32:$filedir\$filename[Magick].png"
    add_custom_command(
            OUTPUT ${IMG}[Trim].png
            COMMAND ${CONVERT} "${IMG}.png" -trim "PNG32:${IMG}[Trim].png"
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            DEPENDS ${IMG}.png
    )
    add_custom_target(
            ${IMGTarget}+Trim ALL
            COMMAND :
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            DEPENDS ${IMG}[Trim].png
    )

    if(EXISTS "${IMG}[Trim].png")
        set(IMGDEPEND ${IMG}[Trim].png)
        if(DEFINED "${IMGTarget}+DIM")
            set(IMGDIM "${${IMGTarget}+DIM}")
        else()
            execute_process(COMMAND ${IDENTIFY} -format "%[fx:w];%[fx:h]" ${IMG}[Trim].png OUTPUT_VARIABLE IMGDIM)
            set("${IMGTarget}+DIM" ${IMGDIM} CACHE INTERNAL "")
        endif()
        list(GET IMGDIM 0 W)
        list(GET IMGDIM 1 H)
        if(${W} GREATER ${H})
            set(S ${W})
        else()
            set(S ${H})
        endif()

        add_custom_command(
                OUTPUT ${IMG}[Square].png
                COMMAND ${CONVERT} "${IMG}[Trim].png" -background transparent -gravity center -resize "${S}x${S}" -extent "${S}x${S}" "PNG32:${IMG}[Square].png"
                WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                DEPENDS ${IMG}[Trim].png
        )
        add_custom_target(
                ${IMGTarget}+Square ALL
                COMMAND :
                WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                DEPENDS ${IMG}[Square].png
        )

        set(small    48)
        set(medium  100)
        set(large   204)
        #set(border   20)
        math(EXPR bsmall  "${small}  - (${small} / 10)")
        math(EXPR bmedium "${medium} - (${medium} / 10)")
        math(EXPR blarge  "${large}  - (${large} / 10)")
        set(
                sizes
                "Favicon"      "0"          "16"         "16x16"                 "16x16"               "center"
                "Small"        "0"          "${small}"   "${small}x${small}"     "${small}x${small}"   "center"
                "Medium"       "${small}"   "${medium}"  "${medium}x${medium}"   "${medium}x${medium}" "center"
                "Large"        "${medium}"  "${large}"   "${large}x${large}"     "${large}x${large}"   "center"
                "Wide"         "${small}"   "${medium}"  "${large}x${medium}"    "${large}x${medium}"  "center"
                "Wide2"        "${medium}"  "${large}"   "${large}"              "${large}x${medium}"  "center"
                "Wide3"        "${medium}"  "${large}"   "${large}"              "${large}x${medium}"  "north"
                "BorderSmall"  "0"          "${bsmall}"  "${bsmall}x${bsmall}"   "${small}x${small}"   "center"
                "BorderMedium" "${bsmall}"  "${bmedium}" "${bmedium}x${bmedium}" "${medium}x${medium}" "center"
                "BorderLarge"  "${bmedium}" "${blarge}"  "${blarge}x${blarge}"   "${large}x${large}"   "center"
                "BorderWide"   "${bsmall}"  "${bmedium}" "${blarge}x${bmedium}"  "${large}x${medium}"  "center"
                "BorderWide2"  "${bmedium}" "${blarge}"  "${blarge}"             "${large}x${medium}"  "center"
                "BorderWide3"  "${bmedium}" "${blarge}"  "${blarge}"             "${large}x${medium}"  "north"
        )
        while(sizes)
            list(POP_FRONT sizes name minBorder minResize resize extent gravity)
            if(${S} GREATER_EQUAL ${minResize})
                # ${CONVERT} "$filedir\$filename$namemod.png" -background transparent -gravity $value[$gravity] -resize $value[$resize] -filter lanczos -extent $value[$extent] "PNG32:$filedir\$filename[$key].png"
                add_custom_command(
                        OUTPUT ${IMG}[${name}].png
                        COMMAND ${CONVERT} "${IMG}[Square].png" -background transparent -gravity ${gravity} -resize "${resize}" -filter lanczos -extent "${extent}" "PNG32:${IMG}[${name}].png"
                        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                        DEPENDS ${IMG}[Square].png
                )
            elseif(${S} GREATER_EQUAL ${minBorder})
                # ${CONVERT} "$filedir\$filename$namemod.png" -background transparent -gravity $value[$gravity]                                         -extent $value[$extent] "PNG32:$filedir\$filename[$key].png"
                add_custom_command(
                        OUTPUT ${IMG}[${name}].png
                        COMMAND ${CONVERT} "${IMG}[Square].png" -background transparent -gravity ${gravity}                                     -extent "${extent}" "PNG32:${IMG}[${name}].png"
                        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                        DEPENDS ${IMG}[Square].png
                )
            endif()
            if(${S} GREATER_EQUAL ${minBorder})
                add_custom_target(
                        ${IMGTarget}+${name} ALL
                        COMMAND :
                        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                        DEPENDS ${IMG}[${name}].png
                )
            endif()
        endwhile ()
    else()
        set(IMGDEPEND RerunCmake)
    endif()
endforeach()

if(REBUILD)

endif()