# lemon--

An experimental fork of the
[lemon parser generator](http://www.hwaci.com/sw/lemon/lemon.html)
that hopes to be more compatible with c++11.  Placement constructors and
destructors are automatically generated, so you shouldn't need to deal with
pointers as much (unless, of course, you want to!)


## contrived example


###lemon--
	%token_type {std::any} // c++17
    %type number_list{std::vector<int>}
    %type string_list{std::vector<std::string>}
    // or use a std::shared_ptr or std::unique_ptr...

    program ::= list.

    list ::= number_list(L). {

    	for (auto i : L) {
    		printf("%d\n", i);
    	}
    }

    list ::= string_list(L). {

        for (const auto &s : L) {
            printf("%s\n", s.c_str());
        }
    }

    number_list(L) ::= NUMBER(N). {
        L.push_back(std::any_cast<int>(N));
    }

    number_list(L) ::= number_list(L) NUMBER(N). {
    	L.push_back(std::any_cast<int>(N));
    }

    string_list(L) ::= STRING(S). {
        L.push_back(std::any_cast<std::string>(S));
    }

    string_list(L) ::= string_list(L) STRING(S). {
        L.push_back(std::any_cast<std::string>(S));
    }


###lemon

	%token_type {std::any *)}
    %token_destructor { delete $$ }

    %type number_list{std::vector<int> *}
    %type string_list{std::vector<std::string> *}
    %destructor number_list { delete $$ }
    %destructor string_list { delete $$ }

    program ::= list.

    list ::= number_list(L). {

    	for (auto i : *L) {
    		printf("%d\n", i);
    	}
    	delete L;
    }

    list ::= string_list(L). {

        for (const auto &s : *L) {
            printf("%s\n", s.c_str());
        }
        delete L;
    }


    number_list(L) ::= NUMBER(N). {
        L = new std::vector<int>();
        L->push_back(std::any_cast<int>(*N));
        delete N;
    }

    number_list(L) ::= number_list(L) NUMBER(N). {
        L->push_back(std::any_cast<int>(*N));
        delete N;
    }

    string_list(L) ::= NUMBER(N). {
        L = new std::vector<std::string>();
        L->push_back(std::any_cast<std::string>(*S));
        delete S;
    }

    string_list(L) ::= string_list(L) STRING(S). {
        L->push_back(std::any_cast<std::string>(*S));
        delete S;
    }

### Notes

I tried to make these examples fairly similar. Lemon-- automatically 
destructs all right-hand-side variables and automatically constructs the
left-hand-side variable.  Additionally, it uses placement constructors and
destructors so you use non-POD data type instead of pointers.

Standard lemon only calls the %destructor for right-hand-side terms that
don't have an alias, hence the manual deletion above.


## Exceptions

Don't do that.  

    one(RHS) ::= two(A). {
        RHS = std::move(A);
        throw std::exception("oops");
    }

In the above code, `RHS`'s destructor will not be called and it will leak.  
`A` will remain on the parse stack in a valid but unspecified state.  Thus
trying to continue parsing may or may not work (additionally, there may be
other parser internals that are out of sync.)  The destructor leak is fixable
but due to the other reasons, you'd be better off remaining unexceptional.


## Other enhancements

There is a new `%header` declaration. This is a block of code that will be
written, verbatim, into the generated header file.

    %header {
        void *ParseAlloc(void *(*mallocProc)(size_t));
        void ParseFree(void *p, void (*freeProc)(void*));
        void Parse(void *yyp, int yymajor, struct MyTokenType *yyminor);
    }

# other versions for your consideration

* http://sourceforge.net/projects/lemonxx/
* http://sourceforge.net/projects/lemonpp/
