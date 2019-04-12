# Makefile for osx-dist conversion using reposurgeon
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
REMOTE_URL = https://smlnj-gforge.cs.uchicago.edu/svn/smlnj/osx-dist
#REMOTE_URL = https://osx-dist.googlecode.com/svn/
# CVS_HOST = osx-dist.cvs.sourceforge.net
#CVS_HOST = cvs.savannah.gnu.org
# CVS_MODULE = osx-dist
#REMOTE_URL = cvs://$(CVS_HOST)/osx-dist\#$(CVS_MODULE)
READ_OPTIONS =
VERBOSITY = "verbose 1"
REPOSURGEON = reposurgeon

# Configuration ends here

.PHONY: local-clobber remote-clobber gitk gc compare clean dist stubmap
# Tell make not to auto-remove tag directories, because it only tries rm
# and hence fails
.PRECIOUS: osx-dist-%-checkout osx-dist-%-git

default: osx-dist-git rmignore gc

# Build the converted repo from the second-stage fast-import stream
osx-dist-git: osx-dist.fi
	rm -fr osx-dist-git; $(REPOSURGEON) 'read <osx-dist.fi' 'prefer git' 'rebuild osx-dist-git'

# Build the second-stage fast-import stream from the first-stage stream dump
osx-dist.fi: osx-dist.svn osx-dist.opts osx-dist.lift osx-dist.map $(EXTRAS)
	$(REPOSURGEON) $(VERBOSITY) 'script osx-dist.opts' "read $(READ_OPTIONS) <osx-dist.svn" 'authors read <osx-dist.map' 'sourcetype svn' 'prefer git' 'script osx-dist.lift' 'legacy write >osx-dist.fo' 'write >osx-dist.fi'

# Build the first-stage stream dump from the local mirror
osx-dist.svn: osx-dist-mirror
	(cd osx-dist-mirror/ >/dev/null; repotool export) >osx-dist.svn

# Build a local mirror of the remote repository
osx-dist-mirror:
	repotool mirror $(REMOTE_URL) osx-dist-mirror

# Make a local checkout of the source mirror for inspection
osx-dist-checkout: osx-dist-mirror
	cd osx-dist-mirror >/dev/null; repotool checkout $(PWD)/osx-dist-checkout

# Make a local checkout of the source mirror for inspection at a specific tag
osx-dist-%-checkout: osx-dist-mirror
	cd osx-dist-mirror >/dev/null; repotool checkout $(PWD)/osx-dist-$*-checkout $*

# Force rebuild of first-stage stream from the local mirror on the next make
local-clobber: clean
	rm -fr osx-dist.fi osx-dist-git *~ .rs* osx-dist-conversion.tar.gz osx-dist-*-git

# Force full rebuild from the remote repo on the next make.
remote-clobber: local-clobber
	rm -fr osx-dist.svn osx-dist-mirror osx-dist-checkout osx-dist-*-checkout

# Get the (empty) state of the author mapping from the first-stage stream
stubmap: osx-dist.svn
	$(REPOSURGEON) "read $(READ_OPTIONS) <osx-dist.svn" 'authors write >osx-dist.map'

# Compare the histories of the unconverted and converted repositories at head
# and all tags.
EXCLUDE = -x CVS -x .svn -x .git
EXCLUDE += -x .svnignore -x .gitignore
headcompare: osx-dist-mirror osx-dist-git
	repotool compare $(EXCLUDE) osx-dist-mirror osx-dist-git
tagscompare: osx-dist-mirror osx-dist-git
	repotool compare-tags $(EXCLUDE) osx-dist-mirror osx-dist-git
branchescompare: osx-dist-mirror osx-dist-git
	repotool compare-branches $(EXCLUDE) osx-dist-mirror osx-dist-git
allcompare: osx-dist-mirror osx-dist-git
	repotool compare-all $(EXCLUDE) osx-dist-mirror osx-dist-git

# General cleanup and utility
clean:
	rm -fr *~ .rs* osx-dist-conversion.tar.gz *.svn *.fi *.fo

# Bundle up the conversion metadata for shipping
SOURCES = Makefile osx-dist.lift osx-dist.map $(EXTRAS)
osx-dist-conversion.tar.gz: $(SOURCES)
	tar --dereference --transform 's:^:osx-dist-conversion/:' -czvf osx-dist-conversion.tar.gz $(SOURCES)

dist: osx-dist-conversion.tar.gz

#
# The following productions are git-specific
#

# Browse the generated git repository
gitk: osx-dist-git
	cd osx-dist-git; gitk --all

# Run a garbage-collect on the generated git repository.  Import doesn't.
# This repack call is the active part of gc --aggressive.  This call is
# tuned for very large repositories.
gc: osx-dist-git
	cd osx-dist-git; time git -c pack.threads=1 repack -AdF --window=1250 --depth=250

rmignore:
	cd *-git && git filter-branch --force --index-filter \
	'git rm --cached --ignore-unmatch .gitignore' \
	--prune-empty --tag-name-filter cat -- --all && cd -
