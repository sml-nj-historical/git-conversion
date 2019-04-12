# Makefile for benchmarks conversion using reposurgeon
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
REMOTE_URL = https://smlnj-gforge.cs.uchicago.edu/svn/smlnj/benchmarks
#REMOTE_URL = https://benchmarks.googlecode.com/svn/
# CVS_HOST = benchmarks.cvs.sourceforge.net
#CVS_HOST = cvs.savannah.gnu.org
# CVS_MODULE = benchmarks
#REMOTE_URL = cvs://$(CVS_HOST)/benchmarks\#$(CVS_MODULE)
READ_OPTIONS =
VERBOSITY = "verbose 1"
REPOSURGEON = reposurgeon

# Configuration ends here

.PHONY: local-clobber remote-clobber gitk gc compare clean dist stubmap
# Tell make not to auto-remove tag directories, because it only tries rm
# and hence fails
.PRECIOUS: benchmarks-%-checkout benchmarks-%-git

default: benchmarks-git rmignore gc

# Build the converted repo from the second-stage fast-import stream
benchmarks-git: benchmarks.fi
	rm -fr benchmarks-git; $(REPOSURGEON) 'read <benchmarks.fi' 'prefer git' 'rebuild benchmarks-git'

# Build the second-stage fast-import stream from the first-stage stream dump
benchmarks.fi: benchmarks.svn benchmarks.opts benchmarks.lift benchmarks.map $(EXTRAS)
	$(REPOSURGEON) $(VERBOSITY) 'script benchmarks.opts' "read $(READ_OPTIONS) <benchmarks.svn" 'authors read <benchmarks.map' 'sourcetype svn' 'prefer git' 'script benchmarks.lift' 'legacy write >benchmarks.fo' 'write >benchmarks.fi'

# Build the first-stage stream dump from the local mirror
benchmarks.svn: benchmarks-mirror
	(cd benchmarks-mirror/ >/dev/null; repotool export) >benchmarks.svn

# Build a local mirror of the remote repository
benchmarks-mirror:
	repotool mirror $(REMOTE_URL) benchmarks-mirror

# Make a local checkout of the source mirror for inspection
benchmarks-checkout: benchmarks-mirror
	cd benchmarks-mirror >/dev/null; repotool checkout $(PWD)/benchmarks-checkout

# Make a local checkout of the source mirror for inspection at a specific tag
benchmarks-%-checkout: benchmarks-mirror
	cd benchmarks-mirror >/dev/null; repotool checkout $(PWD)/benchmarks-$*-checkout $*

# Force rebuild of first-stage stream from the local mirror on the next make
local-clobber: clean
	rm -fr benchmarks.fi benchmarks-git *~ .rs* benchmarks-conversion.tar.gz benchmarks-*-git

# Force full rebuild from the remote repo on the next make.
remote-clobber: local-clobber
	rm -fr benchmarks.svn benchmarks-mirror benchmarks-checkout benchmarks-*-checkout

# Get the (empty) state of the author mapping from the first-stage stream
stubmap: benchmarks.svn
	$(REPOSURGEON) "read $(READ_OPTIONS) <benchmarks.svn" 'authors write >benchmarks.map'

# Compare the histories of the unconverted and converted repositories at head
# and all tags.
EXCLUDE = -x CVS -x .svn -x .git
EXCLUDE += -x .svnignore -x .gitignore
headcompare: benchmarks-mirror benchmarks-git
	repotool compare $(EXCLUDE) benchmarks-mirror benchmarks-git
tagscompare: benchmarks-mirror benchmarks-git
	repotool compare-tags $(EXCLUDE) benchmarks-mirror benchmarks-git
branchescompare: benchmarks-mirror benchmarks-git
	repotool compare-branches $(EXCLUDE) benchmarks-mirror benchmarks-git
allcompare: benchmarks-mirror benchmarks-git
	repotool compare-all $(EXCLUDE) benchmarks-mirror benchmarks-git

# General cleanup and utility
clean:
	rm -fr *~ .rs* benchmarks-conversion.tar.gz *.svn *.fi *.fo

# Bundle up the conversion metadata for shipping
SOURCES = Makefile benchmarks.lift benchmarks.map $(EXTRAS)
benchmarks-conversion.tar.gz: $(SOURCES)
	tar --dereference --transform 's:^:benchmarks-conversion/:' -czvf benchmarks-conversion.tar.gz $(SOURCES)

dist: benchmarks-conversion.tar.gz

#
# The following productions are git-specific
#

# Browse the generated git repository
gitk: benchmarks-git
	cd benchmarks-git; gitk --all

# Run a garbage-collect on the generated git repository.  Import doesn't.
# This repack call is the active part of gc --aggressive.  This call is
# tuned for very large repositories.
gc: benchmarks-git
	cd benchmarks-git; time git -c pack.threads=1 repack -AdF --window=1250 --depth=250

rmignore:
	cd *-git && tail .gitignore && git log | tail && git filter-branch --force --index-filter \
	'git rm --cached --ignore-unmatch .gitignore' \
	--prune-empty --tag-name-filter cat -- --all && cd -
