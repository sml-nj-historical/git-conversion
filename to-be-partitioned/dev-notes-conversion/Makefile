# Makefile for dev-notes conversion using reposurgeon
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
REMOTE_URL = https://smlnj-gforge.cs.uchicago.edu/svn/smlnj/dev-notes
#REMOTE_URL = https://dev-notes.googlecode.com/svn/
# CVS_HOST = dev-notes.cvs.sourceforge.net
#CVS_HOST = cvs.savannah.gnu.org
# CVS_MODULE = dev-notes
#REMOTE_URL = cvs://$(CVS_HOST)/dev-notes\#$(CVS_MODULE)
READ_OPTIONS =
VERBOSITY = "verbose 1"
REPOSURGEON = reposurgeon

# Configuration ends here

.PHONY: local-clobber remote-clobber gitk gc compare clean dist stubmap
# Tell make not to auto-remove tag directories, because it only tries rm
# and hence fails
.PRECIOUS: dev-notes-%-checkout dev-notes-%-git

default: dev-notes-git rmignore gc

# Build the converted repo from the second-stage fast-import stream
dev-notes-git: dev-notes.fi
	rm -fr dev-notes-git; $(REPOSURGEON) 'read <dev-notes.fi' 'prefer git' 'rebuild dev-notes-git'

# Build the second-stage fast-import stream from the first-stage stream dump
dev-notes.fi: dev-notes.svn dev-notes.opts dev-notes.lift dev-notes.map $(EXTRAS)
	$(REPOSURGEON) $(VERBOSITY) 'script dev-notes.opts' "read $(READ_OPTIONS) <dev-notes.svn" 'authors read <dev-notes.map' 'sourcetype svn' 'prefer git' 'script dev-notes.lift' 'legacy write >dev-notes.fo' 'write >dev-notes.fi'

# Build the first-stage stream dump from the local mirror
dev-notes.svn: dev-notes-mirror
	(cd dev-notes-mirror/ >/dev/null; repotool export) >dev-notes.svn

# Build a local mirror of the remote repository
dev-notes-mirror:
	repotool mirror $(REMOTE_URL) dev-notes-mirror

# Make a local checkout of the source mirror for inspection
dev-notes-checkout: dev-notes-mirror
	cd dev-notes-mirror >/dev/null; repotool checkout $(PWD)/dev-notes-checkout

# Make a local checkout of the source mirror for inspection at a specific tag
dev-notes-%-checkout: dev-notes-mirror
	cd dev-notes-mirror >/dev/null; repotool checkout $(PWD)/dev-notes-$*-checkout $*

# Force rebuild of first-stage stream from the local mirror on the next make
local-clobber: clean
	rm -fr dev-notes.fi dev-notes-git *~ .rs* dev-notes-conversion.tar.gz dev-notes-*-git

# Force full rebuild from the remote repo on the next make.
remote-clobber: local-clobber
	rm -fr dev-notes.svn dev-notes-mirror dev-notes-checkout dev-notes-*-checkout

# Get the (empty) state of the author mapping from the first-stage stream
stubmap: dev-notes.svn
	$(REPOSURGEON) "read $(READ_OPTIONS) <dev-notes.svn" 'authors write >dev-notes.map'

# Compare the histories of the unconverted and converted repositories at head
# and all tags.
EXCLUDE = -x CVS -x .svn -x .git
EXCLUDE += -x .svnignore -x .gitignore
headcompare: dev-notes-mirror dev-notes-git
	repotool compare $(EXCLUDE) dev-notes-mirror dev-notes-git
tagscompare: dev-notes-mirror dev-notes-git
	repotool compare-tags $(EXCLUDE) dev-notes-mirror dev-notes-git
branchescompare: dev-notes-mirror dev-notes-git
	repotool compare-branches $(EXCLUDE) dev-notes-mirror dev-notes-git
allcompare: dev-notes-mirror dev-notes-git
	repotool compare-all $(EXCLUDE) dev-notes-mirror dev-notes-git

# General cleanup and utility
clean:
	rm -fr *~ .rs* dev-notes-conversion.tar.gz *.svn *.fi *.fo

# Bundle up the conversion metadata for shipping
SOURCES = Makefile dev-notes.lift dev-notes.map $(EXTRAS)
dev-notes-conversion.tar.gz: $(SOURCES)
	tar --dereference --transform 's:^:dev-notes-conversion/:' -czvf dev-notes-conversion.tar.gz $(SOURCES)

dist: dev-notes-conversion.tar.gz

#
# The following productions are git-specific
#

# Browse the generated git repository
gitk: dev-notes-git
	cd dev-notes-git; gitk --all

# Run a garbage-collect on the generated git repository.  Import doesn't.
# This repack call is the active part of gc --aggressive.  This call is
# tuned for very large repositories.
gc: dev-notes-git
	cd dev-notes-git; time git -c pack.threads=1 repack -AdF --window=1250 --depth=250

rmignore:
	cd *-git && tail .gitignore && git log | tail && git filter-branch --force --index-filter \
	'git rm --cached --ignore-unmatch .gitignore' \
	--prune-empty --tag-name-filter cat -- --all && cd -
