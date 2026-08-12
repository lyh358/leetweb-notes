既在左子树找p和q，又在右子树找p和q。
只要找到p或q的其中一个就可以返回，如果left和right都不为空，说明左右子树各找到p和q当中的一个，那么p和q在root的两侧。
如果left不为空，说明p,q在左子树。
如果right不为空，说明p,q在右子树。
left和right都为空，说明找不到。