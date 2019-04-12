# Makefile for bugs conversion using reposurgeon
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
REMOTE_URL = https://smlnj-gforge.cs.uchicago.edu/svn/smlnj/bugs
#REMOTE_URL = https://bugs.googlecode.com/svn/
# CVS_HOST = bugs.cvs.sourceforge.net
#CVS_HOST = cvs.savannah.gnu.org
# CVS_MODULE = bugs
#REMOTE_URL = cvs://$(CVS_HOST)/bugs\#$(CVS_MODULE)
READ_OPTIONS =
VERBOSITY = "verbose 1"
REPOSURGEON = reposurgeon

# Configuration ends here

.PHONY: local-clobber remote-clobber gitk gc compare clean dist stubmap
# Tell make not to auto-remove tag directories, because it only tries rm
# and hence fails
.PRECIOUS: bugs-%-checkout bugs-%-git

default: bugs-git rmignore gc

# Build the converted repo from the second-stage fast-import stream
bugs-git: bugs.fi
	rm -fr bugs-git; $(REPOSURGEON) 'read <bugs.fi' 'prefer git' 'rebuild bugs-git'

# Build the second-stage fast-import stream from the first-stage stream dump
bugs.fi: bugs.svn bugs.opts bugs.lift bugs.map $(EXTRAS)
	$(REPOSURGEON) $(VERBOSITY) 'script bugs.opts' "read $(READ_OPTIONS) <bugs.svn" 'authors read <bugs.map' 'sourcetype svn' 'prefer git' 'script bugs.lift' 'legacy write >bugs.fo' 'write >bugs.fi'

# Build the first-stage stream dump from the local mirror
bugs.svn: bugs-mirror
	(cd bugs-mirror/ >/dev/null; repotool export) >bugs.svn

# Build a local mirror of the remote repository
bugs-mirror:
	repotool mirror $(REMOTE_URL) bugs-mirror

# Make a local checkout of the source mirror for inspection
bugs-checkout: bugs-mirror
	cd bugs-mirror >/dev/null; repotool checkout $(PWD)/bugs-checkout

# Make a local checkout of the source mirror for inspection at a specific tag
bugs-%-checkout: bugs-mirror
	cd bugs-mirror >/dev/null; repotool checkout $(PWD)/bugs-$*-checkout $*

# Force rebuild of first-stage stream from the local mirror on the next make
local-clobber: clean
	rm -fr bugs.fi bugs-git *~ .rs* bugs-conversion.tar.gz bugs-*-git

# Force full rebuild from the remote repo on the next make.
remote-clobber: local-clobber
	rm -fr bugs.svn bugs-mirror bugs-checkout bugs-*-checkout

# Get the (empty) state of the author mapping from the first-stage stream
stubmap: bugs.svn
	$(REPOSURGEON) "read $(READ_OPTIONS) <bugs.svn" 'authors write >bugs.map'

# Compare the histories of the unconverted and converted repositories at head
# and all tags.
EXCLUDE = -x CVS -x .svn -x .git
EXCLUDE += -x .svnignore -x .gitignore
headcompare: bugs-mirror bugs-git
	repotool compare $(EXCLUDE) bugs-mirror bugs-git
tagscompare: bugs-mirror bugs-git
	repotool compare-tags $(EXCLUDE) bugs-mirror bugs-git
branchescompare: bugs-mirror bugs-git
	repotool compare-branches $(EXCLUDE) bugs-mirror bugs-git
allcompare: bugs-mirror bugs-git
	repotool compare-all $(EXCLUDE) bugs-mirror bugs-git

# General cleanup and utility
clean:
	rm -fr *~ .rs* bugs-conversion.tar.gz *.svn *.fi *.fo

# Bundle up the conversion metadata for shipping
SOURCES = Makefile bugs.lift bugs.map $(EXTRAS)
bugs-conversion.tar.gz: $(SOURCES)
	tar --dereference --transform 's:^:bugs-conversion/:' -czvf bugs-conversion.tar.gz $(SOURCES)

dist: bugs-conversion.tar.gz

#
# The following productions are git-specific
#

# Browse the generated git repository
gitk: bugs-git
	cd bugs-git; gitk --all

# Run a garbage-collect on the generated git repository.  Import doesn't.
# This repack call is the active part of gc --aggressive.  This call is
# tuned for very large repositories.
gc: bugs-git
	cd bugs-git; time git -c pack.threads=1 repack -AdF --window=1250 --depth=250

rmignore:
	cd *-git && tail .gitignore && git log | tail && git filter-branch --force --index-filter \
	'git rm --cached --ignore-unmatch .gitignore' \
	--prune-empty --tag-name-filter cat -- --all && cd -
