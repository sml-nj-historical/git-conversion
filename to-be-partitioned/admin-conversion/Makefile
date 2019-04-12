# Makefile for admin conversion using reposurgeon
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
REMOTE_URL = https://smlnj-gforge.cs.uchicago.edu/svn/smlnj/admin
#REMOTE_URL = https://admin.googlecode.com/svn/
# CVS_HOST = admin.cvs.sourceforge.net
#CVS_HOST = cvs.savannah.gnu.org
# CVS_MODULE = admin
#REMOTE_URL = cvs://$(CVS_HOST)/admin\#$(CVS_MODULE)
READ_OPTIONS =
VERBOSITY = "verbose 1"
REPOSURGEON = reposurgeon

# Configuration ends here

.PHONY: local-clobber remote-clobber gitk gc compare clean dist stubmap
# Tell make not to auto-remove tag directories, because it only tries rm
# and hence fails
.PRECIOUS: admin-%-checkout admin-%-git

default: admin-git rmignore gc

# Build the converted repo from the second-stage fast-import stream
admin-git: admin.fi
	rm -fr admin-git; $(REPOSURGEON) 'read <admin.fi' 'prefer git' 'rebuild admin-git'

# Build the second-stage fast-import stream from the first-stage stream dump
admin.fi: admin.svn admin.opts admin.lift admin.map $(EXTRAS)
	$(REPOSURGEON) $(VERBOSITY) 'script admin.opts' "read $(READ_OPTIONS) <admin.svn" 'authors read <admin.map' 'sourcetype svn' 'prefer git' 'script admin.lift' 'legacy write >admin.fo' 'write >admin.fi'

# Build the first-stage stream dump from the local mirror
admin.svn: admin-mirror
	(cd admin-mirror/ >/dev/null; repotool export) >admin.svn

# Build a local mirror of the remote repository
admin-mirror:
	repotool mirror $(REMOTE_URL) admin-mirror

# Make a local checkout of the source mirror for inspection
admin-checkout: admin-mirror
	cd admin-mirror >/dev/null; repotool checkout $(PWD)/admin-checkout

# Make a local checkout of the source mirror for inspection at a specific tag
admin-%-checkout: admin-mirror
	cd admin-mirror >/dev/null; repotool checkout $(PWD)/admin-$*-checkout $*

# Force rebuild of first-stage stream from the local mirror on the next make
local-clobber: clean
	rm -fr admin.fi admin-git *~ .rs* admin-conversion.tar.gz admin-*-git

# Force full rebuild from the remote repo on the next make.
remote-clobber: local-clobber
	rm -fr admin.svn admin-mirror admin-checkout admin-*-checkout

# Get the (empty) state of the author mapping from the first-stage stream
stubmap: admin.svn
	$(REPOSURGEON) "read $(READ_OPTIONS) <admin.svn" 'authors write >admin.map'

# Compare the histories of the unconverted and converted repositories at head
# and all tags.
EXCLUDE = -x CVS -x .svn -x .git
EXCLUDE += -x .svnignore -x .gitignore
headcompare: admin-mirror admin-git
	repotool compare $(EXCLUDE) admin-mirror admin-git
tagscompare: admin-mirror admin-git
	repotool compare-tags $(EXCLUDE) admin-mirror admin-git
branchescompare: admin-mirror admin-git
	repotool compare-branches $(EXCLUDE) admin-mirror admin-git
allcompare: admin-mirror admin-git
	repotool compare-all $(EXCLUDE) admin-mirror admin-git

# General cleanup and utility
clean:
	rm -fr *~ .rs* admin-conversion.tar.gz *.svn *.fi *.fo

# Bundle up the conversion metadata for shipping
SOURCES = Makefile admin.lift admin.map $(EXTRAS)
admin-conversion.tar.gz: $(SOURCES)
	tar --dereference --transform 's:^:admin-conversion/:' -czvf admin-conversion.tar.gz $(SOURCES)

dist: admin-conversion.tar.gz

#
# The following productions are git-specific
#

# Browse the generated git repository
gitk: admin-git
	cd admin-git; gitk --all

# Run a garbage-collect on the generated git repository.  Import doesn't.
# This repack call is the active part of gc --aggressive.  This call is
# tuned for very large repositories.
gc: admin-git
	cd admin-git; time git -c pack.threads=1 repack -AdF --window=1250 --depth=250

rmignore:
	cd *-git && git lg | tail -2 && git filter-branch --force --index-filter \
	'git rm --cached --ignore-unmatch .gitignore' \
	--prune-empty --tag-name-filter cat -- --all && cd -
