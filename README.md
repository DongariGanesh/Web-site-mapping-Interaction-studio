# Web-site-mapping-Interaction-studio
SalesforceInteractions.init({
    cookieDomain: "shop.revolvesoftech.com",
    consents: [{
        purpose: SalesforceInteractions.mcis.ConsentPurpose.Personalization,
        provider: "Revotrix Consent Manager",
        status: SalesforceInteractions.ConsentStatus.OptIn
    }]
}).then(() => {
    const sitemapConfig = {
        global: {
            onActionEvent: (actionEvent) => {
                const email = SalesforceInteractions.mcis.getValueFromNestedObject("window._etmc.user_info.email");
                if (email) {
                    actionEvent.user = actionEvent.user || {};
                    actionEvent.user.identities = actionEvent.user.identities || {};
                    actionEvent.user.identities.emailAddress = email;
                }
                return actionEvent;
            },
            //Creating Global Content Zones for all page types
            contentZones: [{
                name: "global_infobar_top_of_page"
            }, {
                name: "global_popup",
                selector: "header .site-header"
            }]
        },
        //Creating page types for tracking which page the user is browsing.
        pageTypeDefault: {
            name: "Homepage",
            interaction: {
                name: "Homepage"
            }
        },
        pageTypes: [{
                name: "product_detail",
                isMatch: () => new Promise((resolve, reject) => {
                    let isMatchPDP = setTimeout(() => {
                        return SalesforceInteractions.DisplayUtils.pageElementLoaded(".mf-single-product", "html").then(() => {
                            isMatchPDP = null;
                            resolve(true);
                        })
                    }, 5000);
                    return SalesforceInteractions.DisplayUtils.pageElementLoaded(".mf-single-product", "html").then(() => {
                        clearTimeout(isMatchPDP);
                        isMatchPDP = null;
                        resolve(true);
                    })
                }),
                interaction: {
                    // Getting product-related information from the webpage and building product catalog in Personalization
                    name: SalesforceInteractions.CatalogObjectInteractionName.ViewCatalogObject,
                    catalogObject: {
                        type: "Product",
                        id: () => {
                            return SalesforceInteractions.util.resolveWhenTrue.bind(() => {
                                const productId = SalesforceInteractions.cashDom(".meta-sku .meta-value").text().trim();
                                const products = getProductsFromDataLayer();
                                if (products && products.length > 0) {
                                    return products[0].id;
                                } else if (productId) {
                                    return productId;
                                } else {
                                    return false;
                                }
                            })
                        },
                        attributes: {
                            sku: {
                                id: () => {
                                    return SalesforceInteractions.util.resolveWhenTrue.bind(() => {
                                        return SalesforceInteractions.cashDom(".meta-sku .meta-value").text().trim();
                                    })
                                }
                            },
                            name: Evergage.resolvers.fromSelector('.product_title'),
                            url: SalesforceInteractions.resolvers.fromHref(),
                            imageUrl: SalesforceInteractions.resolvers.fromSelectorAttribute(".wp-post-image", "src"),
                            price: () => {
                                return SalesforceInteractions.resolvers.fromSelector(".price", (price) => parseFloat(price.replace(/[^0-9\.]+/g, "")))(Text)
                            },
                        },
                        relatedCatalogObjects: {
                            Category: SalesforceInteractions.DisplayUtils.pageElementLoaded(".container .product_meta .posted_in a", "html").then((ele) => {
                                return SalesforceInteractions.resolvers.buildCategoryId(".container .product_meta .posted_in a", null, null, (categoryId) => [categoryId.toUpperCase()]);
                            }),
                        }
                    }
                },
                // Adding listeners on click events getting detail of products into the cart.
                listeners: [
                    SalesforceInteractions.listener("click", ".single_add_to_cart_button", () => {
                        let lineItem = SalesforceInteractions.mcis.buildLineItemFromPageState(".cart .quantity .qty-box .qty");
                        SalesforceInteractions.sendEvent({
                            interaction: {
                                name: SalesforceInteractions.CartInteractionName.AddToCart,
                                lineItem: lineItem
                            }
                        })
                    })
                ]
            },
            // Getting Cart Page details from the website
            {
                name: "cart",
                isMatch: () => /\/cart/.test(window.location.href),
                interaction: {
                    name: SalesforceInteractions.CartInteractionName.ReplaceCart,
                    lineItems: SalesforceInteractions.DisplayUtils.pageElementLoaded(".mf-remove, .wc-proceed-to-checkout .checkout-button", "html").then(() => {
                        let cartLineItems = [];
                        SalesforceInteractions.cashDom(".woocommerce-cart-form .cart .cart_item").each((index, ele) => {
                            let itemQuantity = (SalesforceInteractions.cashDom(ele).find(".product-quantity .quantity .qty-box .qty").attr("value"));
                            if (itemQuantity && itemQuantity > 0) {
                                let lineItem = {
                                    catalogObjectType: "Product",
                                    catalogObjectId: SalesforceInteractions.cashDom(ele).find(".product-remove .mf-remove").attr("data-product_sku").trim(),
                                    price: SalesforceInteractions.cashDom(ele).find(".product-subtotal .amount").text().trim().replace(/[^0-9\.]+/g, "") / itemQuantity,
                                    quantity: itemQuantity
                                };
                                cartLineItems.push(lineItem);
                            }
                        })
                        return cartLineItems;
                    })
                }
            },
            // Getting Order Confirmation details from the website
            {
                name: "order_confirmation",
                isMatch: () => new Promise((resolve, reject) => {
                    let isMatchPDP = setTimeout(() => {
                        return SalesforceInteractions.DisplayUtils.pageElementLoaded(".woocommerce-order", "html").then(() => {
                            resolve(true);
                        })
                    }, 1000);
                }),
                interaction: {
                    name: SalesforceInteractions.OrderInteractionName.Purchase,
                    order: {
                        id: SalesforceInteractions.DisplayUtils.pageElementLoaded(".woocommerce-order .order_details .order", "html").then((ele) => {
                            return (SalesforceInteractions.resolvers.fromSelector(".woocommerce-order .order_details .order", (orderid) => parseFloat(orderid.replace(/[^0-9\.]+/g, "")))(Text));
                        }),
                        lineItems: SalesforceInteractions.DisplayUtils.pageElementLoaded(".woocommerce-order-details .order_details .order_item", "html").then(() => {
                            let purchaseLineItems = [];
                            SalesforceInteractions.cashDom(".woocommerce-order-details .order_details .order_item").each((index, ele) => {
                                let itemQuantity = parseInt(SalesforceInteractions.cashDom(ele).find(".product-quantity").text().replace('×', '').trim());
                                if (itemQuantity && itemQuantity > 0) {
                                    let lineItem = {
                                        catalogObjectType: "Product",
                                        catalogObjectId: SalesforceInteractions.cashDom(ele).find(".product-sku").text().trim(),
                                        SalesforceInteractions.cashDom(ele).find(".product-name").text().trim().slice(0, -4),
                                        price: SalesforceInteractions.cashDom(ele).find(".product-total .amount").text().trim().replace(/[^0-9\.]+/g, "") / itemQuantity,
                                        quantity: itemQuantity
                                    };
                                    purchaseLineItems.push(lineItem);
                                }
                            })
                            return purchaseLineItems;
                        })
                    }
                }
            },
            //Getting Login Page details for creating a User profile
            {
                name: "loginpage",
                isMatch: () => /\/my-account/.test(window.location.pathname),
                interaction: {
                    name: "Login"
                },
                listeners: [
                    SalesforceInteractions.listener("click", ".button", () => {
                        const email = SalesforceInteractions.cashDom("#username").val();
                        if (email) {
                            SalesforceInteractions.sendEvent({
                                interaction: {
                                    name: "Logged In"
                                },
                                user: {
                                    identities: {
                                        emailAddress: email
                                    }
                                }
                            });
                        }
                    })
                ]
            }
        ]
    };
    const getProductsFromDataLayer = () => {
        if (window.dataLayer) {
            for (let i = 0; i < window.dataLayer.length; i++) {
                if ((window.dataLayer[i].ecommerce && window.dataLayer[i].ecommerce.detail || {}).products) {
                    return window.dataLayer[i].ecommerce.detail.products;
                }
            }
        }
    };
    SalesforceInteractions.initSitemap(sitemapConfig);
    // Initializing the Sitemap
});
