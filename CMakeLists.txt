### HughAI config
#

# configure_java_skirmish_ai()

	set(myDir                    "${CMAKE_CURRENT_SOURCE_DIR}")
	get_last_path_part(dirName ${myDir})
	set(myName                   "${dirName}")
	set(myLoaderSourceDir        "${myDir}/src-loader")
	set(myAiSourceDir            "${myDir}/src-ai")
	# If main Java package is "my.ai.pkg", this has to be set to "my".
	get_first_sub_dir_name(firstSrcSubDir ${myLoaderSourceDir})
	set(myJavaPkgFirstPart       "${firstSrcSubDir}")

	# Check if the user wants to compile the AI
	if    (BUILD_Java_AIINTERFACE AND NOT myName MATCHES "${AI_EXCLUDE_REGEX}")
		set(BUILD_THIS_SKIRMISHAI TRUE)
	else  (BUILD_Java_AIINTERFACE AND NOT myName MATCHES "${AI_EXCLUDE_REGEX}")
		set(BUILD_THIS_SKIRMISHAI FALSE)
		message("warning: ${myName} Skirmish AI will not be built!")
	endif (BUILD_Java_AIINTERFACE AND NOT myName MATCHES "${AI_EXCLUDE_REGEX}")


	# Compile and install
	if    (BUILD_THIS_SKIRMISHAI)
		# Assemble meta data
		get_version_plus_dep_file(myVersion myVersionDepFile)
		set(myTarget             "${myName}")
		set(myLoaderTarget       "${myTarget}-Loader")
		set(myAiTarget           "${myTarget}-AI")
		set(myLoaderSourceTarget "${myLoaderTarget}-Source")
		set(myAiSourceTarget     "${myAiTarget}-Source")
		set(myDataDir            "${SKIRMISH_AI_DATA}/${myName}/${myVersion}")
		# CMAKE_CURRENT_BINARY_DIR: .../spring-build-dir/AI/Skirmish/${myName}
		set(myBuildDir           "${CMAKE_CURRENT_BINARY_DIR}")
		set(myJavaLoaderBuildDir "${myBuildDir}/classes/loader")
		set(myJavaAiBuildDir     "${myBuildDir}/classes/ai")
		set(myLoaderJarFile      "SkirmishAI")
		set(myAiJarFile          "UnderlyingAI")
		set(myLoaderBinJarFile   "${myLoaderJarFile}.jar")
		set(myAiBinJarFile       "${myAiJarFile}.jar")
		set(myLoaderSrcJarFile   "${myLoaderJarFile}-src.jar")
		set(myAiSrcJarFile       "${myAiJarFile}-src.jar")
		set(myJLibDir            "${myDir}/data/jlib")

		skirmish_ai_message(STATUS "Found Skirmish AI: ${myName} ${myVersion}")

		# Create our full Java class-path
		create_classpath(myJavaLibs ${myJLibDir})
		concat_classpaths(myClassPath "${CLASSPATH_Java_AIINTERFACE}" "${myJavaLibs}" ${myAiSourceDir} ${myLoaderSourceDir} ${JavaOO_AIWRAPPER_JAR_CLASSPATH})

		# Create a list of all the AIs source files (for compiling)
		file(GLOB_RECURSE myLoaderSources RELATIVE "${myLoaderSourceDir}" FOLLOW_SYMLINKS "${myLoaderSourceDir}/*.java")
		file(GLOB_RECURSE myAiSources     RELATIVE "${myAiSourceDir}"     FOLLOW_SYMLINKS "${myAiSourceDir}/*.java")
		# Create a list of all the AIs source files (for dependency tracking)
		file(GLOB_RECURSE myLoaderSourcesDep FOLLOW_SYMLINKS "${myLoaderSourceDir}/*.java")
		file(GLOB_RECURSE myAiSourcesDep     FOLLOW_SYMLINKS "${myAiSourceDir}/*.java")

		# Assemble project generated libraries we depend on
		set(myDependTargets  "${Java_AIINTERFACE_TARGET}"  "${JavaOO_AIWRAPPER_TARGET}")
		set(myDependLibFiles "${Java_AIINTERFACE_JAR_BIN}" "${JavaOO_AIWRAPPER_JAR_BIN}")
		set_source_files_properties(${myDependLibFiles} PROPERTIES GENERATED TRUE)

		add_custom_target(${myLoaderTarget}       DEPENDS ${myBuildDir}/${myLoaderBinJarFile})
		set_source_files_properties(${myBuildDir}/${myLoaderBinJarFile} PROPERTIES GENERATED TRUE)
		add_dependencies(${myLoaderTarget} ${myDependTargets})

		add_custom_target(${myLoaderSourceTarget} DEPENDS ${myBuildDir}/${myLoaderSrcJarFile})
		set_source_files_properties(${myBuildDir}/${myLoaderSrcJarFile} PROPERTIES GENERATED TRUE)

		add_custom_target(${myAiTarget}           DEPENDS ${myBuildDir}/${myAiBinJarFile})
		set_source_files_properties(${myBuildDir}/${myAiBinJarFile} PROPERTIES GENERATED TRUE)
		add_dependencies(${myAiTarget} ${myDependTargets})

		add_custom_target(${myAiSourceTarget}     DEPENDS ${myBuildDir}/${myAiSrcJarFile})
		set_source_files_properties(${myBuildDir}/${myAiSrcJarFile} PROPERTIES GENERATED TRUE)

		add_custom_target(${myTarget} ALL DEPENDS ${myVersionDepFile})
		add_dependencies(${myTarget} ${myLoaderTarget} ${myAiTarget} ${myLoaderSourceTarget} ${myAiSourceTarget})

		# Compile the loader library
		add_custom_command(
			OUTPUT
				"${myBuildDir}/${myLoaderBinJarFile}"
			COMMAND "${CMAKE_COMMAND}"
				"-E" "make_directory" "${myJavaLoaderBuildDir}"
			COMMAND "${Java_JAVAC_EXECUTABLE}"
				"${JAVA_COMPILE_FLAG_CONDITIONAL}"
				"-cp" "${myClassPath}"
				"-d" "${myJavaLoaderBuildDir}"
				${myLoaderSources}
			COMMAND "${Java_JAR_EXECUTABLE}"
				"cmf" "${myDir}/manifest.mf" "${myBuildDir}/${myLoaderBinJarFile}"
				"-C" "${myJavaLoaderBuildDir}" "${myJavaPkgFirstPart}"
			DEPENDS
				${myDependLibFiles}
				${myLoaderSourcesDep}
			WORKING_DIRECTORY
				"${myLoaderSourceDir}"
			COMMENT
				"  ${myTarget}: Creating library ${myLoaderBinJarFile}" VERBATIM
			)

		# Compile the AI library
		add_custom_command(
			OUTPUT
				"${myBuildDir}/${myAiBinJarFile}"
			COMMAND "${CMAKE_COMMAND}"
				"-E" "make_directory" "${myJavaAiBuildDir}"
			COMMAND "${Java_JAVAC_EXECUTABLE}"
				"${JAVA_COMPILE_FLAG_CONDITIONAL}"
				"-cp" "${myClassPath}"
				"-d" "${myJavaAiBuildDir}"
				${myAiSources}
			COMMAND "${Java_JAR_EXECUTABLE}"
				"cmf" "${myDir}/manifest.mf" "${myBuildDir}/${myAiBinJarFile}"
				"-C" "${myJavaAiBuildDir}" "${myJavaPkgFirstPart}"
			DEPENDS
				${myDependLibFiles}
				${myAiSourcesDep}
			WORKING_DIRECTORY
				"${myAiSourceDir}"
			COMMENT
				"  ${myTarget}: Creating library ${myAiBinJarFile}" VERBATIM
			)

		# Pack the loader sources
		add_custom_command(
			OUTPUT
				"${myBuildDir}/${myLoaderSrcJarFile}"
			COMMAND "${Java_JAR_EXECUTABLE}"
				"cf" "${myBuildDir}/${myLoaderSrcJarFile}"
				"-C" "${myLoaderSourceDir}" "${myJavaPkgFirstPart}"
			DEPENDS
				${myLoaderSourcesDep}
			WORKING_DIRECTORY
				"${myLoaderSourceDir}"
			COMMENT
				"  ${myTarget}: Creating sources archive ${myLoaderSrcJarFile}" VERBATIM
			)

		# Pack the AI sources
		add_custom_command(
			OUTPUT
				"${myBuildDir}/${myAiSrcJarFile}"
			COMMAND "${Java_JAR_EXECUTABLE}"
				"cf" "${myBuildDir}/${myAiSrcJarFile}"
				"-C" "${myAiSourceDir}" "${myJavaPkgFirstPart}"
			DEPENDS
				${myAiSourcesDep}
			WORKING_DIRECTORY
				"${myAiSourceDir}"
			COMMENT
				"  ${myTarget}: Creating sources archive ${myAiSrcJarFile}" VERBATIM
			)

		# Install the data files
		install(DIRECTORY "${myDir}/data/"                      DESTINATION ${myDataDir})
		# Install the library
		install (FILES    "${myBuildDir}/${myLoaderBinJarFile}" DESTINATION ${myDataDir})
		# Install the library
		install (FILES    "${myBuildDir}/${myAiBinJarFile}"     DESTINATION ${myDataDir})
		# Install the sources archive
		install (FILES    "${myBuildDir}/${myLoaderSrcJarFile}" DESTINATION ${myDataDir}/jlib)
		# Install the sources archive
		install (FILES    "${myBuildDir}/${myAiSrcJarFile}"     DESTINATION ${myDataDir}/jlib)
		# Install the Java OO Wrapper libraries
		install(FILES "${JavaOO_AIWRAPPER_JAR_BIN}"             DESTINATION ${myDataDir}/jlib)
		set(wrapperJLibDir "${CMAKE_SOURCE_DIR}/AI/Wrappers/JavaOO/jlib")
		if    (EXISTS ${wrapperJLibDir})
			install(DIRECTORY "${wrapperJLibDir}/"              DESTINATION ${myDataDir}/jlib)
		endif (EXISTS ${wrapperJLibDir})
	endif (BUILD_THIS_SKIRMISHAI)
