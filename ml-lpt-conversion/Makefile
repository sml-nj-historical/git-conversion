# Makefile for ml-lpt conversion using reposurgeon
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
REMOTE_URL = https://smlnj-gforge.cs.uchicago.edu/svn/ml-lpt
#REMOTE_URL = https://ml-lpt.googlecode.com/svn/
# CVS_HOST = ml-lpt.cvs.sourceforge.net
#CVS_HOST = cvs.savannah.gnu.org
# CVS_MODULE = ml-lpt
#REMOTE_URL = cvs://$(CVS_HOST)/ml-lpt\#$(CVS_MODULE)
READ_OPTIONS = --nobranch
VERBOSITY = "verbose 1"
REPOSURGEON = reposurgeon

# Configuration ends here

.PHONY: local-clobber remote-clobber gitk gc compare clean dist stubmap
# Tell make not to auto-remove tag directories, because it only tries rm
# and hence fails
.PRECIOUS: ml-lpt-%-checkout ml-lpt-%-git

default: ml-lpt-git

# Build the converted repo from the second-stage fast-import stream
ml-lpt-git: ml-lpt.fi
	rm -fr ml-lpt-git; $(REPOSURGEON) 'read <ml-lpt.fi' 'prefer git' 'rebuild ml-lpt-git'

# Build the second-stage fast-import stream from the first-stage stream dump
ml-lpt.fi: ml-lpt.svn ml-lpt.opts ml-lpt.lift ml-lpt.map $(EXTRAS)
	vim ml-lpt.svn +230463 &&	$(REPOSURGEON) $(VERBOSITY) 'script ml-lpt.opts' "read $(READ_OPTIONS) <ml-lpt.svn" 'authors read <ml-lpt.map' 'sourcetype svn' 'prefer git' 'script ml-lpt.lift' 'legacy write >ml-lpt.fo' 'write >ml-lpt.fi'

# Build the first-stage stream dump from the local mirror
ml-lpt.svn: ml-lpt-mirror
	(cd ml-lpt-mirror/ >/dev/null; repotool export) | repocutter pathrename '^trunk/' '' >ml-lpt.svn

# Build a local mirror of the remote repository
ml-lpt-mirror:
	repotool mirror $(REMOTE_URL) ml-lpt-mirror

# Make a local checkout of the source mirror for inspection
ml-lpt-checkout: ml-lpt-mirror
	cd ml-lpt-mirror >/dev/null; repotool checkout $(PWD)/ml-lpt-checkout

# Make a local checkout of the source mirror for inspection at a specific tag
ml-lpt-%-checkout: ml-lpt-mirror
	cd ml-lpt-mirror >/dev/null; repotool checkout $(PWD)/ml-lpt-$*-checkout $*

# Force rebuild of first-stage stream from the local mirror on the next make
local-clobber: clean
	rm -fr ml-lpt.fi ml-lpt-git *~ .rs* ml-lpt-conversion.tar.gz ml-lpt-*-git

# Force full rebuild from the remote repo on the next make.
remote-clobber: local-clobber
	rm -fr ml-lpt.svn ml-lpt-mirror ml-lpt-checkout ml-lpt-*-checkout

# Get the (empty) state of the author mapping from the first-stage stream
stubmap: ml-lpt.svn
	$(REPOSURGEON) "read $(READ_OPTIONS) <ml-lpt.svn" 'authors write >ml-lpt.map'

# Compare the histories of the unconverted and converted repositories at head
# and all tags.
EXCLUDE = -x CVS -x .svn -x .git
EXCLUDE += -x .svnignore -x .gitignore
headcompare: ml-lpt-mirror ml-lpt-git
	repotool compare $(EXCLUDE) ml-lpt-mirror ml-lpt-git
tagscompare: ml-lpt-mirror ml-lpt-git
	repotool compare-tags $(EXCLUDE) ml-lpt-mirror ml-lpt-git
branchescompare: ml-lpt-mirror ml-lpt-git
	repotool compare-branches $(EXCLUDE) ml-lpt-mirror ml-lpt-git
allcompare: ml-lpt-mirror ml-lpt-git
	repotool compare-all $(EXCLUDE) ml-lpt-mirror ml-lpt-git

# General cleanup and utility
clean:
	rm -fr *~ .rs* ml-lpt-conversion.tar.gz *.svn *.fi *.fo

# Bundle up the conversion metadata for shipping
SOURCES = Makefile ml-lpt.lift ml-lpt.map $(EXTRAS)
ml-lpt-conversion.tar.gz: $(SOURCES)
	tar --dereference --transform 's:^:ml-lpt-conversion/:' -czvf ml-lpt-conversion.tar.gz $(SOURCES)

dist: ml-lpt-conversion.tar.gz

#
# The following productions are git-specific
#

# Browse the generated git repository
gitk: ml-lpt-git
	cd ml-lpt-git; gitk --all

# Run a garbage-collect on the generated git repository.  Import doesn't.
# This repack call is the active part of gc --aggressive.  This call is
# tuned for very large repositories.
gc: ml-lpt-git
	cd ml-lpt-git; time git -c pack.threads=1 repack -AdF --window=1250 --depth=250
