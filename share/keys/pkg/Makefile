SUBDIR=	trusted

# CoorvoOS repository public key; the private key never goes in the tree.
.if exists(${.CURDIR}/coorvo-repo.pub)
PACKAGE=	pkg-bootstrap
FILES=	coorvo-repo.pub
FILESDIR=	${SHAREDIR}/keys/pkg
FILESMODE=	644
.endif

.include <bsd.prog.mk>
