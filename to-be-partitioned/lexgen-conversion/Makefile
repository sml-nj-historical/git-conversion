# Makefile for lexgen conversion using reposurgeon
#
# Steps to using this:
# 1. Make sure reposurgeon and repotool are on your $PATH.
#    For large repositories It is usually best to run reposurgeon using
#    PyPy: set REPOSURGEON to "pypy" followed by an absolute pathname
#    to reposurgeon.
# 2. (Skip this step if you're starting from a stream file.) For svn, set
#    REMOTE_URL to point at the remote repository you want to convert.
#    If the repository is already in a DVCS such as hg or git,
#    set REMOTE_URL to either the normal cloning URL (starting with hg://,
#    git://, etc.) or to the path of a local clone.
# 3. For cvs, set CVS_HOST to the repo hostname and CVS_MODULE to the module,
#    then uncomment the line that builds REMOTE_URL
#    Note: for CVS hosts other than Sourceforge or Savannah you will need to
#    include the path to the CVS modules directory after the hostname.
# 4. Set any required read options, such as --user-ignores or --nobranch,
#    by setting READ_OPTIONS.
# 5. Run 'make stubmap' to create a stub author map.
# 6. (Optional) set REPOSURGEON to point at a faster cython build of the tool.
# 7. Run 'make' to build a converted repository.
#
# The reason both first- and second-stage stream files are generated is that,
# especially with Subversion, making the first-stage stream file is often
# painfully slow. By splitting the process, we lower the overhead of
# experiments with the lift script.
#
# For a production-quality conversion you will need to edit the map
# file and the lift script.  During the process you can set EXTRAS to
# name extra metadata such as a comments mailbox.
#
# Afterwards, you can use the headcompare and tagscompare productions
# to check your work.
#

EXTRAS =
REMOTE_URL = https://smlnj-gforge.cs.uchicago.edu/svn/smlnj/lexgen
#REMOTE_URL = https://lexgen.googlecode.com/svn/
# CVS_HOST = lexgen.cvs.sourceforge.net
#CVS_HOST = cvs.savannah.gnu.org
# CVS_MODULE = lexgen
#REMOTE_URL = cvs://$(CVS_HOST)/lexgen\#$(CVS_MODULE)
READ_OPTIONS =
VERBOSITY = "verbose 1"
REPOSURGEON = reposurgeon

# Configuration ends here

.PHONY: local-clobber remote-clobber gitk gc compare clean dist stubmap
# Tell make not to auto-remove tag directories, because it only tries rm
# and hence fails
.PRECIOUS: lexgen-%-checkout lexgen-%-git

default: lexgen-git rmignore gc

# Build the converted repo from the second-stage fast-import stream
lexgen-git: lexgen.fi
	rm -fr lexgen-git; $(REPOSURGEON) 'read <lexgen.fi' 'prefer git' 'rebuild lexgen-git'

# Build the second-stage fast-import stream from the first-stage stream dump
lexgen.fi: lexgen.svn lexgen.opts lexgen.lift lexgen.map $(EXTRAS)
	$(REPOSURGEON) $(VERBOSITY) 'script lexgen.opts' "read $(READ_OPTIONS) <lexgen.svn" 'authors read <lexgen.map' 'sourcetype svn' 'prefer git' 'script lexgen.lift' 'legacy write >lexgen.fo' 'write >lexgen.fi'

# Build the first-stage stream dump from the local mirror
lexgen.svn: lexgen-mirror
	(cd lexgen-mirror/ >/dev/null; repotool export) \
	>lexgen.svn

# Build a local mirror of the remote repository
lexgen-mirror:
	repotool mirror $(REMOTE_URL) lexgen-mirror

# Make a local checkout of the source mirror for inspection
lexgen-checkout: lexgen-mirror
	cd lexgen-mirror >/dev/null; repotool checkout $(PWD)/lexgen-checkout

# Make a local checkout of the source mirror for inspection at a specific tag
lexgen-%-checkout: lexgen-mirror
	cd lexgen-mirror >/dev/null; repotool checkout $(PWD)/lexgen-$*-checkout $*

# Force rebuild of first-stage stream from the local mirror on the next make
local-clobber: clean
	rm -fr lexgen.fi lexgen-git *~ .rs* lexgen-conversion.tar.gz lexgen-*-git

# Force full rebuild from the remote repo on the next make.
remote-clobber: local-clobber
	rm -fr lexgen.svn lexgen-mirror lexgen-checkout lexgen-*-checkout

# Get the (empty) state of the author mapping from the first-stage stream
stubmap: lexgen.svn
	$(REPOSURGEON) "read $(READ_OPTIONS) <lexgen.svn" 'authors write >lexgen.map'

# Compare the histories of the unconverted and converted repositories at head
# and all tags.
EXCLUDE = -x CVS -x .svn -x .git
EXCLUDE += -x .svnignore -x .gitignore
headcompare: lexgen-mirror lexgen-git
	repotool compare $(EXCLUDE) lexgen-mirror lexgen-git
tagscompare: lexgen-mirror lexgen-git
	repotool compare-tags $(EXCLUDE) lexgen-mirror lexgen-git
branchescompare: lexgen-mirror lexgen-git
	repotool compare-branches $(EXCLUDE) lexgen-mirror lexgen-git
allcompare: lexgen-mirror lexgen-git
	repotool compare-all $(EXCLUDE) lexgen-mirror lexgen-git

# General cleanup and utility
clean:
	rm -fr *~ .rs* lexgen-conversion.tar.gz *.svn *.fi *.fo

# Bundle up the conversion metadata for shipping
SOURCES = Makefile lexgen.lift lexgen.map $(EXTRAS)
lexgen-conversion.tar.gz: $(SOURCES)
	tar --dereference --transform 's:^:lexgen-conversion/:' -czvf lexgen-conversion.tar.gz $(SOURCES)

dist: lexgen-conversion.tar.gz

#
# The following productions are git-specific
#

# Browse the generated git repository
gitk: lexgen-git
	cd lexgen-git; gitk --all

# Run a garbage-collect on the generated git repository.  Import doesn't.
# This repack call is the active part of gc --aggressive.  This call is
# tuned for very large repositories.
gc: lexgen-git
	cd lexgen-git; time git -c pack.threads=1 repack -AdF --window=1250 --depth=250

rmignore:
	cd *-git && git log | tail && git filter-branch --force --index-filter \
	'git rm --cached --ignore-unmatch .gitignore' \
	--prune-empty --tag-name-filter cat -- --all && cd -
