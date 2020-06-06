	project(banking_lib)

	if (NOT TARGET libbanking)
	    add_library(libbanking STATIC
		${CMAKE_CURRENT_LIST_DIR}/Account.cpp
		${CMAKE_CURRENT_LIST_DIR}/Transaction.cpp
	    )

	    install(TARGETS libbanking
		ARCHIVE DESTINATION lib
		LIBRARY DESTINATION lib
	    )
	endif(NOT TARGET libbanking)

	include_directories(${CMAKE_CURRENT_LIST_DIR})
